# Actividad 3.2 – Seguridad en Nginx ## Autenticación, Control de Acceso y HTTPS 
Este documento describe el proceso completo de instalación, configuración y validación de los cuatro sitios Nginx requeridos en la actividad 3.2:
- portal.seguro.local
- dev.seguro.local
- api.seguro.local
- admin.seguro.local
---
## Descripción de los sitios 
- **portal.seguro.local**
  Portal público con área `/admin` protegida con HTTPS y autenticación HTTP Basic.
- **dev.seguro.local**
  Entorno de desarrollo en puerto 8080, restringido por IP + autenticación (satisfy any) y con cabeceras personalizadas.
- **api.seguro.local**
  API HTTPS obligatoria, respuestas JSON simuladas y rate limiting.
- **admin.seguro.local**
  Panel de administración en HTTPS, solo accesible desde localhost, con autenticación, rate limiting agresivo y HSTS.
---
## 1. Configuración de red 
### 1.1. VirtualBox Se han configurado dos adaptadores de red en la VM Ubuntu: 
**Adaptador 1** 
- Conectado a: NAT
- Propósito: salida a Internet para actualizaciones y apt
- **Adaptador 2**
- Conectado a: Host-only Adapter (VirtualBox Host-Only Ethernet Adapter)
- Propósito: red de pruebas entre host y VM (192.168.56.0/24)
Captura recomendada: pantalla de configuración de red de la VM mostrando Adaptador 1 = NAT y Adaptador 2 = Host-only.
---
### 1.2. Netplan en Ubuntu Se configura enp0s3 (NAT) por DHCP y enp0s8 (Host-only) con IP estática: 
```yaml
network: version: 2 renderer: networkd ethernets: enp0s3: dhcp4: true # NAT para Internet enp0s8: dhcp4: no addresses: - 192.168.56.10/24 
```
Se aplica con: 
```bash 
sudo netplan apply
ip a show enp0s8
``` 
La VM se puede alcanzar desde el host (192.168.56.1) y viceversa. 
--- 
## 2. Dependencias e instalación 
### 2.1. Paquetes 
```bash 
sudo apt update
sudo apt install -y nginx apache2-utils
``` 
- nginx: servidor web
- apache2-utils: comando htpasswd
---
### 2.2. Estructura de directorios
  ```bash
  sudo mkdir -p /var/www/act3_2/{portal,dev,api,admin}
  sudo mkdir -p /etc/nginx/certs
  ```
  ```bash
  sudo cp -r www/portal/* /var/www/act3_2/portal/
  sudo cp -r www/dev/* /var/www/act3_2/dev/
  sudo cp -r www/api/* /var/www/act3_2/api/
  sudo cp -r www/admin/* /var/www/act3_2/admin/
  ```
  ```bash
  sudo chown -R www-data:www-data /var/www/act3_2/{portal,dev,api,admin}
  sudo chmod -R 755 /var/www/act3_2/{portal,dev,api,admin}
  ```
  ---
## 3. Autenticación HTTP Basic 
```bash
sudo htpasswd -bc /etc/nginx/.htpasswd-portal admin 'Admin123!'
sudo htpasswd -bc /etc/nginx/.htpasswd-dev developer 'Dev2025!'
sudo htpasswd -b /etc/nginx/.htpasswd-dev tester 'Test2025!'
sudo htpasswd -bc /etc/nginx/.htpasswd-admin superadmin 'SuperSecure2025!' 
```
--- 
## 4. Certificados SSL 
```bash 
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \ -keyout /etc/nginx/certs/portal.seguro.local.key \ -out /etc/nginx/certs/portal.seguro.local.crt \ -subj "/C=ES/ST=Pontevedra/L=Pontevedra/O=DAW/OU=Portal/CN=portal.seguro.local"
```
```bash 
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \ -keyout /etc/nginx/certs/api.seguro.local.key \ -out /etc/nginx/certs/api.seguro.local.crt \ -subj "/C=ES/ST=Pontevedra/L=Pontevedra/O=DAW/OU=API/CN=api.seguro.local"
```
```bash 
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \ -keyout /etc/nginx/certs/admin.seguro.local.key \ -out /etc/nginx/certs/admin.seguro.local.crt \ -subj "/C=ES/ST=Madrid/L=Madrid/O=DAW/OU=Admin/CN=admin.seguro.local"
```
---
## 5. Rate limiting global 
```
nginx limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s; limit_req_zone $binary_remote_addr zone=admin:10m rate=3r/m;
``` 
---
## 6. Habilitación de sitios 
```bash 
sudo ln -sf /etc/nginx/sites-available/*.conf /etc/nginx/sites-enabled/ sudo nginx -t && sudo systemctl reload nginx
``` 
---
## 7. /etc/hosts 
```
text 127.0.0.1 portal.seguro.local 127.0.0.1 api.seguro.local 127.0.0.1 admin.seguro.local 127.0.0.1 dev.seguro.local
``` 
---
## 8. Pruebas automáticas 
```bash 
chmod +x test.sh
./test.sh
```

<img width="1717" height="631" alt="image" src="https://github.com/user-attachments/assets/9384d9f2-80e8-4026-93aa-819da9ea6416" />

---
## 9. Estructura final del repositorio 
```
text act3_2_nginx_seguridad/
├── README.md
├── test.sh
├── www/
├── nginx/
└── certs/
```

## 10. Pruebas manuales 

Público accesible por HTTP 
```bash
curl -I http://portal.seguro.local/
```
HTTP/1.1 200 OK 
Admin redirige a HTTPS
```bash
curl -I http://portal.seguro.local/admin/
```
HTTP/1.1 301 Moved Permanently
Location: https://portal.seguro.local/admin/ 
Admin requiere autenticación
```bash 
curl -I https://portal.seguro.local/admin/
```
HTTP/1.1 401 Unauthorized 
Admin con credenciales válidas
```bash
curl -I -u admin:Admin123! https://portal.seguro.local/admin/
``` 
HTTP/1.1 200 OK

<img width="1717" height="932" alt="image" src="https://github.com/user-attachments/assets/17a198f8-5fa9-47f6-a123-00e062dc0489" />
---

Desde IP permitida (192.168.56.1): acceso sin credenciales 
```bash
curl -I http://192.168.56.10:8080/
```
HTTP/1.1 200 OK
X-Environment: Development 
Desde IP no permitida sin credenciales: denegado
```bash
curl -I http://dev.seguro.local:8080/ (desde otra red)
C:\Users\Pablo>curl -I http://dev.seguro.local:8080/
curl: (6) Could not resolve host: dev.seguro.local
```
HTTP/1.1 401 Unauthorized 
Desde IP no permitida CON credenciales: permitido
```bash
curl -I -u developer:Dev2025! http://dev.seguro.local:8080/
```
HTTP/1.1 200 OK

<img width="1719" height="297" alt="image" src="https://github.com/user-attachments/assets/8262e6ef-2b67-4010-81b1-f3b9aff0282f" />
---

HTTP redirige a HTTPS 
```bash
curl -I http://api.seguro.local/
```
HTTP/1.1 301 Moved Permanently
Location: https://api.seguro.local/ 
Endpoints JSON funcionan
```bash
curl https://api.seguro.local/v1/status
```
{"status":"online","version":"1.0","timestamp":"2025-01-15T10:30:00Z"} 
Rate limiting funciona (11 peticiones rápidas)
```bash
for i in {1..30}; do curl -I https://api.seguro.local/health; done
```
Primeras 21 peticiones: 200 OK
Resto: 429 Too Many Requests

<img width="1716" height="846" alt="image" src="https://github.com/user-attachments/assets/510c3507-1d20-4224-9369-c91ed2b0f9ec" />
---

HTTP devuelve 444 (conexión cerrada) 
```bash
curl -I http://admin.seguro.local/
```
curl: (52) Empty reply from server 
Desde localhost sin credenciales: denegado
```bash
curl -I https://localhost/ -H "Host: admin.seguro.local" --insecure
```
HTTP/1.1 401 Unauthorized 
Desde localhost con credenciales: permitido
```bash
curl -I https://localhost/ -H "Host: admin.seguro.local" -u superadmin:SuperSecure2025! -insecure
```
HTTP/1.1 200 OK
Strict-Transport-Security: max-age=31536000; includeSubDomains 
Desde otra IP (aunque tenga credenciales): denegado
```bash
curl -I https://admin.seguro.local/ -u superadmin:SuperSecure2025! --insecure
```
HTTP/1.1 403 Forbidden

<img width="1719" height="752" alt="image" src="https://github.com/user-attachments/assets/91fea200-06a6-4d09-bb34-47e89eb2744d" />
---


1. Portal público: Navegador mostrando página de inicio
<img width="1720" height="1326" alt="image" src="https://github.com/user-attachments/assets/78612d53-b8f7-49ca-9448-560ae23e2b9f" />


2. Portal admin: Diálogo de autenticación
<img width="1720" height="1318" alt="image" src="https://github.com/user-attachments/assets/99f89455-964c-4bc4-a51a-cec1e3781f09" />


3. Portal admin: Página admin con candado HTTPS verde
<img width="1719" height="1325" alt="image" src="https://github.com/user-attachments/assets/d9b61e1c-caf2-4606-922a-bdc22bb8e37f" />


4. Dev: Navegador accediendo sin credenciales (desde IP permitida)
<img width="1720" height="1320" alt="image" src="https://github.com/user-attachments/assets/2917d23a-6094-4bae-a759-06172d81cca2" />


5. API: Respuesta JSON en navegador/Postman
<img width="1718" height="1320" alt="image" src="https://github.com/user-attachments/assets/5c2b5309-a819-47ad-b554-f677f160ea02" />


6. Rate limiting: Terminal mostrando respuestas 429


7. Admin: Página accesible solo desde localhost
<img width="1714" height="394" alt="image" src="https://github.com/user-attachments/assets/e5e9bd14-6520-48cf-8a7a-8d49a7c5ee1d" />


8. Logs de Nginx: Accesos registrados mostrando IP real (192.168.56.1 desde Windows)
<img width="1718" height="618" alt="image" src="https://github.com/user-attachments/assets/13666871-c3c0-4337-8f05-ae44b91d9b6f" />


9. Configuración Nginx: Archivo .conf de uno de los sitios
<img width="1717" height="1104" alt="image" src="https://github.com/user-attachments/assets/cafa83fc-5241-4b1d-a104-c094df7bb593" />


10. Configuración de red: Captura de VirtualBox mostrando los dos adaptadores (NAT + Host-Only)
<img width="823" height="516" alt="image" src="https://github.com/user-attachments/assets/fbd8ac4c-e064-43f4-ab8d-4fbd24361785" />

<img width="822" height="518" alt="image" src="https://github.com/user-attachments/assets/ec06c73d-8970-4e5d-8e85-aa57ef8ec5e3" />

