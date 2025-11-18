# Proyecto Final - Infraestructura Computacional

## Información del Proyecto
- **Estudiantes:** Diego, Brahian, Sara
- **Materia:** Infraestructura Computacional
- **Fecha:** Noviembre 2025

## Objetivo
Implementar una solución de virtualización basada en contenedores con almacenamiento confiable mediante RAID y LVM, incluyendo monitorización con Netdata.

---

## 1. Configuración de RAID

### 1.1 Discos utilizados
Se utilizaron 9 discos de 5GB cada uno:
- **RAID 1 #1 (md0):** sdb, sdc, sdd
- **RAID 1 #2 (md1):** sde, sdf, sdg
- **RAID 1 #3 (md2):** sdh, sdi, sdj

### 1.2 Comandos para crear los RAID
```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
sudo mdadm --create /dev/md1 --level=1 --raid-devices=3 /dev/sde /dev/sdf /dev/sdg
sudo mdadm --create /dev/md2 --level=1 --raid-devices=3 /dev/sdh /dev/sdi /dev/sdj
```

### 1.3 Verificación
```bash
cat /proc/mdstat
```

---

## 2. Configuración de LVM

### 2.1 Crear Physical Volumes
```bash
sudo pvcreate /dev/md0
sudo pvcreate /dev/md1
sudo pvcreate /dev/md2
```

### 2.2 Crear Volume Group
```bash
sudo vgcreate vg__project /dev/md0 /dev/md1 /dev/md2
```

### 2.3 Crear Logical Volumes
```bash
sudo lvcreate -L 1G -n lv_apache vg__project
sudo lvcreate -L 1G -n lv_mysql vg__project
sudo lvcreate -L 1G -n lv_nginx vg__project
```

### 2.4 Formatear los volúmenes
```bash
sudo mkfs.ext4 /dev/vg__project/lv_apache
sudo mkfs.ext4 /dev/vg__project/lv_mysql
sudo mkfs.ext4 /dev/vg__project/lv_nginx
```

### 2.5 Montar los volúmenes
```bash
sudo mount /dev/vg__project/lv_apache /home/brahian/apache
sudo mount /dev/vg__project/lv_mysql /home/brahian/mysql
sudo mount /dev/vg__project/lv_nginx /home/brahian/nginx
```

### 2.6 Configurar montaje automático (/etc/fstab)
```
/dev/vg__project/lv_apache  /home/brahian/apache  ext4  defaults  0  2
/dev/vg__project/lv_mysql   /home/brahian/mysql   ext4  defaults  0  2
/dev/vg__project/lv_nginx   /home/brahian/nginx   ext4  defaults  0  2
```

---

## 3. Contenedores Docker

### 3.1 Dockerfile para Apache
```dockerfile
FROM httpd:latest

EXPOSE 80
```

### 3.2 Dockerfile para MySQL
```dockerfile
FROM mysql:latest

ENV MYSQL_ROOT_PASSWORD=mi_password_seguro

EXPOSE 3306
```

### 3.3 Dockerfile para Nginx
```dockerfile
FROM nginx:latest

EXPOSE 80
```

### 3.4 Construir las imágenes
```bash
docker build -t mi_apache ~/proyecto-final/dockerfiles/apache
docker build -t mi_mysql ~/proyecto-final/dockerfiles/mysql
docker build -t mi_nginx ~/proyecto-final/dockerfiles/nginx
```

### 3.5 Crear los contenedores
```bash
docker run -d --name apache_server -p 8080:80 -v /home/brahian/apache:/usr/local/apache2/htdocs mi_apache
docker run -d --name mysql_server -p 3306:3306 -v /home/brahian/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mi_password_seguro mi_mysql
docker run -d --name nginx_server -p 8081:80 -v /home/brahian/nginx:/usr/share/nginx/html mi_nginx
```

---

## 4. Pruebas de Funcionamiento

### 4.1 Acceso a Apache
- URL: http://localhost:8080
- Resultado: Página "Hola desde Apache - Proyecto Final"

### 4.2 Acceso a Nginx
- URL: http://localhost:8081
- Resultado: Página "Hola desde Nginx - Proyecto Final"

### 4.3 MySQL
- Base de datos creada: proyecto_final
- Tabla: estudiantes
- Datos de prueba insertados correctamente

### 4.4 Prueba de Persistencia
Se eliminaron y recrearon los contenedores, verificando que los datos persisten en los volúmenes LVM.

---

## 5. Monitorización con Netdata

### 5.1 Crear contenedor Netdata
```bash
docker run -d --name netdata \
  -p 19999:19999 \
  --cap-add SYS_PTRACE \
  --security-opt apparmor=unconfined \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /etc/passwd:/host/etc/passwd:ro \
  -v /etc/group:/host/etc/group:ro \
  -v /etc/os-release:/host/etc/os-release:ro \
  netdata/netdata
```

### 5.2 Acceso al Dashboard
- URL: http://localhost:19999
- Se pueden visualizar los 4 contenedores en tiempo real

---

## 6. Arquitectura Final

```
9 Discos (sdb-sdj)
    ↓
3 RAID 1 (md0, md1, md2)
    ↓
1 Volume Group (vg__project)
    ↓
3 Logical Volumes (lv_apache, lv_mysql, lv_nginx)
    ↓
3 Carpetas montadas (/home/brahian/apache, mysql, nginx)
    ↓
4 Contenedores Docker (Apache, MySQL, Nginx, Netdata)
```

---

## 7. Conclusiones

- Se implementó exitosamente una infraestructura de almacenamiento redundante con RAID 1
- LVM permite flexibilidad para gestionar el espacio de almacenamiento
- Los bind mounts de Docker garantizan la persistencia de datos
- Netdata proporciona monitorización en tiempo real de todos los servicios