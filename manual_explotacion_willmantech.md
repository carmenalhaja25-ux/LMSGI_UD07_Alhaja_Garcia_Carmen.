# Manual de Explotación Tecnológica - ERP WillmanTech S.L.

Este documento técnico es la guía oficial de explotación, despliegue y mantenimiento preventivo para la infraestructura integrada de gestión empresarial (ERP/CRM) de la organización WillmanTech S.L. La estructura, redacción y diseño de este manual han sido elaborados con las directrices de la norma internacional ISO/IEC/IEEE 26514:2022.

## 1. Introducción y Arquitectura del Sistema

El ERP está montado sobre Odoo 16 (utilizando los módulos de account, sale y product) y utiliza PostgreSQL 15 como base de datos. Todo funciona aislado dentro de una red de Docker (bridge).

### Topología Lógica y Orquestación

A continuación se detalla la especificación del archivo de orquestación `docker-compose.yml` que define el entorno productivo:

```yaml
version: '3.8'

networks:
  willmantech_network:
    driver: bridge

services:
  db:
    image: postgres:15
    container_name: willmantech_db
    environment:
      - POSTGRES_DB=odoo_prod
      - POSTGRES_USER=odoo_admin
      - POSTGRES_PASSWORD=willman_secure_pass_2026
    volumes:
      - willmantech_db_data:/var/lib/postgresql/data
    networks:
      - willmantech_network
    restart: always

  web:
    image: odoo:16.0
    container_name: willmantech_erp
    depends_on:
      - db
    ports:
      - "8069:8069"
    environment:
      - HOST=db
      - USER=odoo_admin
      - PASSWORD=willman_secure_pass_2026
    volumes:
      - willmantech_web_data:/var/lib/odoo
      - ./custom_modules:/mnt/extra-addons
    networks:
      - willmantech_network
    restart: always

volumes:
  willmantech_db_data:
  willmantech_web_data:
```

# Guía de Instalación y Reinstalación del Entorno.

1. Clonar el repositorio y entrar a la carpeta:

git clone [https://github.com/carmenalhaja25-ux/LMSGI_UD07_Alhaja_Garc-a_Carmen.git](https://github.com/carmenalhaja25-ux/LMSGI_UD07_Alhaja_Garc-a_Carmen.git)
cd LMSGI_UD07_Alhaja_Garc-a_Carmen 

2. Comprobar que el puerto 8069 está libre:

netstat -an | grep 8069

3. Construir y levantar los contenedores en segundo plano:

docker-compose up -d --build

4. Ver los logs en tiempo real para verificar que todo ha arrancado bien:

docker-compose logs -f 

5. Configuración inicial: Entra en http://localhost:8069, crea la base de datos e instala los módulos de ventas y contabilidad.

# Seguridad y Control de Acceso

Aplicamos el principio de menor privilegio para proteger los datos de la empresa mediante tres perfiles de usuario perfectamente definidos. En primer lugar, el Admin del Sistema cuenta con control total sobre la infraestructura (permisos CRUD completos), encargándose de la gestión de usuarios y de las plantillas QWeb sin ningún tipo de restricción. Por otro lado, el Responsable Contable tiene acceso restringido a los módulos de contabilidad y facturación para gestionar el flujo transaccional, quedando completamente excluido de la configuración del servidor, del entorno Docker y de la modificación de impuestos. Finalmente, el Agente Comercial opera únicamente en la fase inicial del ciclo de ventas, lo que le permite crear presupuestos y gestionar la ficha de clientes, pero tiene bloqueado el acceso para validar facturas, realizar asientos en el libro mayor o ver la contabilidad global de la empresa.

Política de contraseñas:
Fuerza: Mínimo 12 caracteres (Mayúsculas, minúsculas, números y símbolos).

Rotación: Cambiar obligatoriamente cada 180 días. No se pueden repetir las últimas 4 contraseñas.

# Procedimiento de Backup y Restauración.

Para garantizar la continuidad de negocio ante incidentes de pérdida de datos o corrupción lógica del almacenamiento persistente, se define un protocolo síncrono de respaldo que procesa de manera independiente la información estructurada y los datos binarios no estructurados.

## Procedimiento de Respaldo (Backup).

1. Backup de la Base de Datos (.dump)

docker exec -t willmantech_db pg_dump -U odoo_admin -F c -b -v -f /var/lib/postgresql/data/backup_willmantech_$(date +%Y%m%d).dump odoo_prod

2. Backup del Filestore (Imágenes/Adjuntos)

tar -czvf backup_filestore_willmantech_$(date +%Y%m%d).tar.gz /var/lib/docker/volumes/lmsgi_ud07_willmantech_web_data/_data

# Flujo Operativo de Facturación e Informes.

Extracción (ORM): Odoo busca en la base de datos los datos de la cabecera de la factura (account.move) y sus líneas correspondientes (account.move.line).

Procesamiento QWeb: El motor de Odoo lee el archivo XML de la plantilla. Procesa los bucles (t-foreach) y los condicionales (como t-if="has_discount" para saber si muestra o no la columna de descuentos). Al terminar, genera un archivo HTML con CSS embebido.

Conversión a PDF: Odoo le pasa ese código HTML a una herramienta del sistema llamada wkhtmltopdf (un navegador WebKit sin interfaz gráfica). Este programa calcula los márgenes y genera el archivo PDF final que se descarga en la computadora del cliente.