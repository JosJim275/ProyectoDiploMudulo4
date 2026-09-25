# Troubleshooting — Harbor sobre Kubernetes

Todos los problemas de este documento ocurrieron de verdad durante la construcción del proyecto, en un cluster de 3 nodos Rocky Linux 9.7 (kubeadm + Calico + MetalLB + cert-manager + Harbor). Se documentan con su causa real y la solución que funcionó, no de forma genérica.

## Resumen (tabla problema → solución)

| # | Problema | Causa | Solución rápida |
|---|---|---|---|
| 1 | `dnf` no descarga el repo de Docker CE | DNS no resuelve nombres | Fijar `nameserver` en `/etc/resolv.conf` |
| 2 | Falla instalación de Helm: falta `tar` | Instalación mínima de Rocky sin `tar` | `dnf install -y tar` antes de Helm |
| 3 | Helm "no encontrado" tras instalarse bien | `$PATH` de Ansible no incluye `/usr/local/bin` | Añadir `environment: PATH` a las tareas |
| 4 | `IPAddressPool` de MetalLB rechazado (`no route to host`) | El webhook del controller aún no estaba listo | Esperar con `kubectl wait` antes de aplicar |
| 5 | Calico: `no matches for kind "Installation"` | Los CRDs del operador aún no se habían registrado | Espera activa (`until`/`retries`) sobre el CRD |
| 6 | Nodos nunca quedan `Ready` aunque el playbook no marca error | Condición de idempotencia mal diseñada saltó pasos necesarios | Verificar el recurso real, no el namespace |
| 7 | `kernel: soft lockup` en el control plane | RAM insuficiente para el control plane + Calico | Subir RAM de la VM (mínimo 4 GB) |
| 8 | `curl`/`kubectl` a un pod en otro nodo: `no route to host` | El firewall no confiaba en la red de los propios nodos | Añadir la red de los nodos a la zona `trusted` |
| 9 | `ERROR! Attempting to decrypt but no vault secrets found` | Ansible carga `group_vars/all/` completo, incluido el vault cifrado | Usar siempre `--ask-vault-pass` |
| 10 | Un worker queda `NotReady` tras reiniciar las VMs | La VM se apagó o congeló al restaurar el snapshot | Reiniciar la VM y esperar su recuperación |

## Detalle, comandos y enlaces

### 1. `dnf` no puede descargar el repositorio de Docker CE
**Error:**
```
Failed to download metadata for repo 'docker-ce-stable': Cannot download repomd.xml: All mirrors were tried
```
**Diagnóstico:**
```bash
ping -c 2 8.8.8.8                    # ¿hay salida a internet?
ping -c 2 download.docker.com        # ¿resuelve nombres?
cat /etc/resolv.conf                 # ¿qué DNS está configurado?
```
Si el primer `ping` responde pero el segundo no, el problema es de DNS, no de conectividad.

**Solución paso a paso:**
1. Verificar el DNS activo con los comandos de arriba.
2. Fijar un servidor DNS funcional:
   ```bash
   echo 'nameserver 8.8.8.8' | sudo tee /etc/resolv.conf
   ```
3. Volver a intentar la descarga o re-ejecutar el playbook (es idempotente, retoma donde falló).
4. Para que sobreviva a reinicios, fijarlo vía NetworkManager en vez del archivo directo:
   ```bash
   nmcli con mod "<nombre-conexion>" ipv4.dns "8.8.8.8"
   ```

**Documentación oficial:** [Rocky Linux Docs — Network Configuration](https://docs.rockylinux.org/) · [NetworkManager DNS](https://networkmanager.dev/docs/api/latest/)

---

### 2. Falla la instalación de Helm por falta de `tar`
**Error:**
```
[ERROR] Could not find tar. It is required to extract the helm binary archive.
```
**Diagnóstico:**
```bash
which tar || echo "tar no está instalado"
```
**Solución paso a paso:**
1. Instalar el paquete faltante:
   ```bash
   sudo dnf install -y tar
   ```
2. Volver a ejecutar el script/playbook de instalación de Helm.

**Documentación oficial:** [Helm — Installing Helm](https://helm.sh/docs/intro/install/)

---

### 3. Helm "no encontrado" aunque sí se instaló
**Error:**
```
helm not found. Is /usr/local/bin on your $PATH?
Failed to install helm
```
pero `ls /usr/local/bin/helm` muestra que el binario sí existe.

**Diagnóstico:**
```bash
ls -la /usr/local/bin/helm
sudo /usr/local/bin/helm version
echo $PATH
```
Si el binario existe y responde con `sudo` pero el error persiste en Ansible, el `$PATH` de la tarea no incluye `/usr/local/bin`.

**Solución paso a paso:**
1. En la tarea de Ansible, declarar el `PATH` explícitamente:
   ```yaml
   - name: Instalar Helm
     ansible.builtin.shell: curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
     environment:
       PATH: "/usr/local/bin:{{ ansible_env.PATH }}"
     failed_when: false
   - name: Confirmar instalación real
     ansible.builtin.stat:
       path: /usr/local/bin/helm
     register: helm_bin
     failed_when: not helm_bin.stat.exists
   ```
2. Para el usuario normal: `echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc && source ~/.bashrc`

**Documentación oficial:** [Ansible — environment keyword](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_environment.html)

---

### 4. MetalLB rechaza el `IPAddressPool`: webhook no listo
**Error:**
```
failed calling webhook "ipaddresspoolvalidationwebhook.metallb.io": dial tcp ...: connect: no route to host
```
**Diagnóstico:**
```bash
kubectl get pods -n metallb-system -o wide
kubectl describe pod -n metallb-system -l app.kubernetes.io/component=controller
```
Revisar en los eventos si hay `FailedMount` recientes (el pod aún está montando su certificado).

**Solución paso a paso:**
1. Esperar explícitamente a que el pod esté listo antes de aplicar el pool:
   ```bash
   kubectl wait --for=condition=Ready pod \
     -n metallb-system -l app.kubernetes.io/component=controller --timeout=120s
   ```
2. Reintentar la aplicación del manifiesto:
   ```bash
   kubectl apply -f metallb-pool.yaml
   ```
3. En Ansible, envolver el `apply` en `until`/`retries` con al menos 15 intentos.

**Documentación oficial:** [MetalLB — Configuration](https://metallb.universe.tf/configuration/) · [MetalLB — Installation troubleshooting](https://metallb.universe.tf/installation/)

---

### 5. Calico: `no matches for kind "Installation"` (CRDs no registrados)
**Error:**
```
no matches for kind "Installation" in version "operator.tigera.io/v1": ensure CRDs are installed first
```
**Diagnóstico:**
```bash
kubectl get crd installations.operator.tigera.io
kubectl get pods -n tigera-operator
```
**Solución paso a paso:**
1. Instalar primero el operador y esperar a que registre sus tipos:
   ```bash
   kubectl apply --server-side -f tigera-operator.yaml
   until kubectl get crd installations.operator.tigera.io >/dev/null 2>&1; do sleep 5; done
   ```
2. Recién entonces aplicar `custom-resources.yaml`.

**Documentación oficial:** [Calico — Troubleshooting](https://docs.tigera.io/calico/latest/operations/troubleshoot/) · [Kubernetes — Custom Resource Definitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/)

---

### 6. Los nodos nunca quedan `Ready` pese a que el playbook "termina bien"
**Síntoma:** `ansible-playbook` corre sin `failed`, pero `kubectl get nodes` sigue mostrando `NotReady` indefinidamente.

**Diagnóstico:**
```bash
kubectl get installation.operator.tigera.io default
kubectl get pods -n calico-system
```
Si el recurso `Installation` no existe, Calico nunca se aplicó, aunque el playbook "pasara".

**Causa real de este proyecto:** una tarea de idempotencia verificaba si existía el *namespace* del operador en vez del recurso `Installation`. Como el namespace ya existía de una corrida anterior parcial, **todo el bloque de instalación se saltó**, incluida la tarea pendiente.

**Solución paso a paso:**
1. Cambiar la condición de idempotencia al recurso que realmente importa:
   ```yaml
   - name: ¿Ya está aplicado Calico?
     ansible.builtin.command: kubectl get installation.operator.tigera.io default
     register: calico_check
     failed_when: false
     changed_when: false
   ```
2. Usar `calico_check.rc != 0` como condición para las tareas de instalación.

**Documentación oficial:** [Ansible — Idempotency and check mode](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html)

---

### 7. `kernel: soft lockup` — el control plane se queda sin RAM
**Error:**
```
kernel: watchdog: BUG: soft lockup - CPU#0 stuck for 27s! [containerd-shim]
```
**Diagnóstico:**
```bash
free -h
uptime
```
Un `load average` muy por encima del número de CPUs, combinado con poca RAM `available` y `0B` de swap, confirma sobrecarga de memoria.

**Solución paso a paso:**
1. Apagar la VM: `sudo poweroff`.
2. En el hipervisor, subir la RAM asignada (mínimo 4 GB para el control plane con Calico + cert-manager + MetalLB encima).
3. Encender la VM y esperar 3-5 minutos antes de operar el cluster.
4. Confirmar con `free -h` y `uptime` que la carga baja antes de continuar.

**Documentación oficial:** [Kubernetes — Node resource requirements (kubeadm)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) · [Red Hat — Diagnosing soft lockups](https://access.redhat.com/solutions/18005)

---

### 8. `no route to host` entre el master y un pod en otro nodo
**Error:**
```
curl: (7) Failed to connect to <IP-de-pod> port 9443: No route to host
```
pese a que `ip route get <IP-de-pod>` muestra una ruta válida.

**Diagnóstico:**
```bash
ip route get <IP-de-pod>
sudo firewall-cmd --zone=trusted --list-sources
sudo firewall-cmd --info-zone=trusted
```
**Causa:** el tráfico que sale directamente de un nodo (no de un pod) hacia otro nodo usa como origen la IP real del nodo, no una IP de la red de pods. La zona `trusted` de firewalld solo confiaba en `pod_cidr` y `service_cidr`.

**Solución paso a paso:**
1. Agregar la red de los propios nodos como fuente confiable:
   ```bash
   sudo firewall-cmd --zone=trusted --add-source=<red-de-nodos>/24 --permanent
   sudo firewall-cmd --reload
   ```
2. Repetir en los tres nodos (o vía Ansible con `ansible.posix.firewalld` y un `loop`).
3. Verificar: `curl -k --connect-timeout 5 https://<IP-de-pod>:<puerto>` ya no debe dar `no route to host`.

**Documentación oficial:** [firewalld — Documentation](https://firewalld.org/documentation/) · [Kubernetes — Network Policies and traffic](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

---

### 9. `ERROR! Attempting to decrypt but no vault secrets found`
**Contexto:** aparece incluso en comandos que no usan ninguna variable del vault.

**Diagnóstico:**
```bash
ls group_vars/all/           # confirmar que vault.yml está ahí
ansible-vault view group_vars/all/vault.yml   # probar el descifrado directo
```
**Causa:** Ansible carga automáticamente **todo** `group_vars/all/` al arrancar, incluido cualquier archivo cifrado, sin importar si la tarea lo necesita.

**Solución paso a paso:**
1. Pasar siempre la bandera en cualquier comando de Ansible sobre este proyecto:
   ```bash
   ansible-playbook sitio.yml --ask-vault-pass
   ```
2. Alternativa para no escribirla cada vez (borrar el archivo al terminar la sesión):
   ```bash
   echo 'contraseña' > ~/.vault_pass && chmod 600 ~/.vault_pass
   ansible-playbook sitio.yml --vault-password-file ~/.vault_pass
   rm ~/.vault_pass   # al terminar
   ```

**Documentación oficial:** [Ansible — Vault Guide](https://docs.ansible.com/ansible/latest/vault_guide/index.html)

---

### 10. Un worker queda `NotReady` después de reiniciar las VMs (desde snapshot)
**Error:**
```
ssh: connect to host <IP-worker> port 22: No route to host
```
y `kubectl get nodes` muestra ese nodo como `NotReady`.

**Diagnóstico:**
```bash
kubectl get nodes
ping -c 3 <IP-worker>
```
Si ninguno responde, el problema es de la VM, no de Kubernetes.

**Solución paso a paso:**
1. Revisar el estado de la VM en el hipervisor (apagada, congelada).
2. Encenderla o forzar un reset si está congelada.
3. Esperar 2-3 minutos a que arranquen `kubelet`, `containerd` y Calico.
4. Confirmar:
   ```bash
   kubectl get nodes -w    # Ctrl+C cuando todos digan Ready
   ```

**Documentación oficial:** [Kubernetes — Node status conditions](https://kubernetes.io/docs/concepts/architecture/nodes/#condition) · [Kubeadm — Troubleshooting](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)

## Lección general

La mayoría de los incidentes del #4 en adelante comparten la misma causa raíz: **reiniciar de golpe un stack con muchos componentes satura momentáneamente el nodo más cargado**, produciendo errores que parecen de configuración pero son de sincronización. La respuesta que mejor funcionó siempre fue la misma: esperar 3-5 minutos tras encender las VMs, y diagnosticar con `uptime`/`free -h` antes de asumir que el código está mal.
