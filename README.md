# Proyecto Final - DataExplorer

Este proyecto automatiza la creación de un laboratorio de monitoreo compuesto por dos máquinas virtuales levantadas con Vagrant/VirtualBox y aprovisionadas con Ansible:

- **webserver**: expone un sitio Nginx, publica `/metrics` y aloja Node Exporter.
- **monitoring**: corre Prometheus y Grafana, ya configurados para recopilar las métricas del webserver.

> Ambos playbooks son idempotentes y detectan automáticamente la arquitectura de la VM para descargar los binarios adecuados (amd64 o arm64).

## Estructura del repositorio

```
.
├── ansible/
│   ├── monitoring.yml   # Playbook para Prometheus + Grafana
│   └── webserver.yml    # Playbook para Nginx + Node Exporter
├── vagrantfile          # Define las dos VMs y sus provisioners
└── README.md
```

## Requisitos

- VirtualBox 7.x (o compatible).
- Vagrant 2.3+.
- Acceso a internet para descargar paquetes y releases de Prometheus/Node Exporter/Grafana.

## Puesta en marcha

```bash
git clone https://github.com/morenotechnology/proyecto_final_SO.git
cd proyecto_final_SO
vagrant up            # crea ambas VMs y ejecuta los playbooks
```

Los playbooks pueden relanzarse en cualquier momento con:

```bash
vagrant provision webserver
vagrant provision monitoring
```

## Servicios expuestos

| Servicio        | Host (desde el equipo físico) | Host interno (red privada) | Notas |
|-----------------|-------------------------------|----------------------------|-------|
| Sitio Nginx     | http://localhost:8080         | http://192.168.56.10       | Landing page con datos de la VM |
| Node Exporter   | http://localhost:9100/metrics | http://192.168.56.10:9100  | También disponible vía proxy en `/metrics` del puerto 8080 |
| Prometheus UI   | http://localhost:9090         | http://192.168.56.11:9090  | Targets ya definidos (Prometheus, Node Exporter, Nginx status) |
| Grafana         | http://localhost:3000         | http://192.168.56.11:3000  | Credenciales por defecto `admin / admin` (se recomienda cambiarlas) |

## Qué se automatiza

### `ansible/webserver.yml`
- Instala Nginx y despliega la landing page.
- Descarga Node Exporter según arquitectura y crea el servicio systemd.
- Configura Nginx para exponer `/metrics` y `/nginx_status`.
- Verifica que los puertos 80 y 9100 queden disponibles.

### `ansible/monitoring.yml`
- Instala Prometheus, genera su configuración y valida con `promtool`.
- Crea/activa el servicio systemd.
- Instala Grafana desde el repositorio oficial, lo abre a todas las interfaces y valida su salud.
- Configura automáticamente el datasource “Prometheus” apuntando al webserver.

## Verificación rápida

1. Abrir `http://localhost:8080` y confirmar que se muestra la página de DataExplorer.
2. Consultar `http://localhost:9100/metrics` para revisar métricas crudas.
3. Entrar a `http://localhost:9090/targets` y comprobar que los endpoints están **UP**.
4. Ingresar a Grafana (`http://localhost:3000`, `admin/admin`) y crear un dashboard usando el datasource “Prometheus”.

## Pruebas adicionales sugeridas

- Ejecutar cargas sintéticas (JMeter, k6, loader.io, etc.) contra `http://localhost:8080` para observar el comportamiento en Prometheus/Grafana.
- Cambiar las credenciales por defecto de Grafana y habilitar https si el laboratorio se expone fuera del entorno local.

## Problemas comunes

- **Los servicios fallan en Apple Silicon/arm64**: vuelve a ejecutar `vagrant provision`. Los playbooks ya descargan los binarios `linux-arm64` automáticamente.
- **Puertos ocupados**: asegúrate de que ninguna otra aplicación use 3000/8080/9090/9100 en tu host.
- **Cambios manuales en las VMs**: si se rompe la configuración, basta con `vagrant provision` o destruir/recrear (`vagrant destroy -f && vagrant up`).

---

¡Listo! Con esto cuentas con un entorno reproducible para practicar administración y monitoreo con Vagrant + Ansible.
