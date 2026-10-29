# CaC para AAP 2.6 con credenciales en Azure Key Vault

Configuración como código para Ansible Automation Platform 2.6 en la que
**ningún secreto se guarda en Git**, ni cifrado ni en claro. Las credenciales se
declaran sin password y AAP resuelve cada valor sensible contra Azure Key Vault
cuando un job lo necesita. Se aplica con `infra.aap_configuration`.

## Estructura

```
gobierno/         organizaciones, equipos y definiciones de bootstrap
  organizations.yml
  bootstrap.yml
  source_credentials.yml   documentación de la conexión con Azure (no se aplica)
torre-linux/      una carpeta por torre: credenciales sin password + input sources,
torre-windows/    proyectos, inventarios y job templates
torre-cloud/
playbooks/        bootstrap.yml, site_gobierno.yml, site_torre.yml
ee-cac/           Containerfile del Execution Environment
```

Qué carga cada playbook:

| Playbook | Ficheros | Cuándo |
|---|---|---|
| `bootstrap.yml` | `gobierno/organizations.yml`, `gobierno/bootstrap.yml` | Una vez, a mano: puesta en marcha o recuperación |
| `site_gobierno.yml` | `gobierno/organizations.yml` | En cada push, primer nodo del workflow |
| `site_torre.yml` | los cuatro ficheros de `torre-<x>/` | En cada push, las tres torres en paralelo |

`gobierno/source_credentials.yml` no lo carga ningún playbook. Los cambios en
`gobierno/bootstrap.yml` tampoco los aplica el workflow: hay que relanzar el
bootstrap.

## Cómo funciona

En el repositorio, una credencial declara solo lo que no es secreto, y un
*input source* indica de qué secreto de Key Vault sale cada campo sensible:

```yaml
controller_credentials:
  - name: "Linux Produccion"
    organization: "Torre-Linux"
    credential_type: "Machine"
    inputs:
      username: "ansible-svc"          # sin password

controller_credential_input_sources:
  - target_credential: "Linux Produccion"
    input_field_name: "password"
    source_credential: "Azure Key Vault Source"
    metadata:
      secret_field: "linux-prod-password"   # nombre del secreto en Key Vault
      secret_version: ""                    # vacío = versión vigente
```

`Azure Key Vault Source` es la conexión de AAP con Azure: una credencial del
tipo `Microsoft Azure Key Vault`, que viene de serie en AAP, con los datos de un
service principal de solo lectura. Se crea a mano (ver `gobierno/source_credentials.yml`).

Cuando un job usa `Linux Produccion`, AAP se conecta a Key Vault con esa
identidad, pide `linux-prod-password` y usa el valor para la conexión. El valor
no se guarda en la base de datos de AAP.

Un push a la rama principal dispara el webhook y AAP ejecuta el workflow
`CaC - Despliegue Completo`:

```
push → webhook → sync del proyecto → workflow

   ┌─────────────────────────────┐
   │ CaC - Gobierno              │  playbooks/site_gobierno.yml
   │ (organizaciones y equipos)  │
   └──────────────┬──────────────┘
                  │ on success
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  Torre        Torre        Torre       playbooks/site_torre.yml
  Linux        Windows      Cloud       (extra var torre: torre-<x>)
```

Gobierno va primero porque las torres necesitan que sus organizaciones existan.
Los job templates del CaC solo llevan la credencial `CaC - AAP API`: el
despliegue nunca lee ninguna contraseña de Key Vault, solo declara los enlaces.

## Secretos en Azure Key Vault

| Secreto | Credencial en AAP | Campos | Torre |
|---|---|---|---|
| `linux-prod-password` | Linux Produccion | `password`, `become_password` | Linux |
| `linux-pre-password` | Linux Pre-Produccion | `password`, `become_password` | Linux |
| `oracle-db-password` | Oracle DB Produccion | `password` | Linux |
| `windows-prod-password` | Windows Produccion | `password` | Windows |
| `windows-pre-password` | Windows Pre-Produccion | `password` | Windows |
| `vcenter-password` | vCenter Corporativo | `password` | Cloud |

Los inventarios y proyectos de las torres son de ejemplo.

## Puesta en marcha

**En Azure**

1. Crear el Key Vault y los secretos de la tabla anterior.
2. Crear un service principal para AAP y darle **solo** `get` y `list` sobre los
   secretos: con access policies, o con el rol *Key Vault Secrets User* si el
   vault usa RBAC.

**En AAP**

3. Ajustar en `gobierno/bootstrap.yml`: `bootstrap_cac_repo` y `bootstrap_ee_image`.
4. Construir `ee-cac/Containerfile` y publicarlo en un registro accesible desde
   AAP. Las colecciones van horneadas en la imagen.
5. Crear a mano, en la organización Default:
   - la credencial `Azure Key Vault Source`, como se describe en
     `gobierno/source_credentials.yml`;
   - el tipo de credencial `CaC - AAP API`, con la definición de
     `gobierno/bootstrap.yml`, y una credencial `CaC - AAP API` de ese tipo con
     la URL de AAP y una cuenta con permisos para aplicar la configuración;
   - la credencial SCM `GitHub - CaC Read Only` (deploy key de solo lectura);
   - el EE `EE CaC Dispatch`, el inventario `CaC - Localhost` y el proyecto
     `CaC - Azure Key Vault`. El bootstrap los vuelve a declarar y los mantiene.
6. Crear y lanzar el JT `CaC - Bootstrap`: playbook `playbooks/bootstrap.yml`,
   inventario `CaC - Localhost`, EE `EE CaC Dispatch` y credencial
   `CaC - AAP API`. Crea organizaciones, equipos, los job templates del CaC y el
   workflow.
7. Lanzar una vez el workflow `CaC - Despliegue Completo` y registrar el webhook
   del repositorio con la URL y la clave que muestra su pestaña Details, evento
   `push`.

## Alta de una credencial

1. Quien gestiona Key Vault crea el secreto, por ejemplo `linux-backup-password`,
   y comunica solo su nombre.
2. El equipo de la torre declara la credencial y su input source en
   `torre-linux/credentials.yml`:

   ```yaml
   controller_credentials:
     - name: "Linux Backup"
       organization: "Torre-Linux"
       credential_type: "Machine"
       inputs:
         username: "ansible-backup-svc"

   controller_credential_input_sources:
     - target_credential: "Linux Backup"
       input_field_name: "password"
       source_credential: "Azure Key Vault Source"
       metadata:
         secret_field: "linux-backup-password"
         secret_version: ""
   ```

3. Pull request y merge. El workflow crea la credencial.

El nombre de la credencial tiene que coincidir exactamente en los dos bloques.
Quien escribe y quien revisa el cambio ven qué secreto se usa, pero no su valor.
Si el secreto no existe en Key Vault, la credencial se crea igual pero falla al
resolverse: el botón **Test** de su *input source* lo detecta.

## Rotación

Se crea una versión nueva del secreto en Key Vault. Con `secret_version: ""`,
AAP usa la nueva a partir del siguiente job, sin tocar el repositorio ni AAP.

Rotar en Key Vault **no cambia la contraseña en los servidores**: solo cambia el
valor que usa AAP. Las dos cosas tienen que ir juntas, normalmente con un job
que cambie la contraseña en los sistemas y actualice Key Vault a la vez.

## Notas

- `infra.aap_configuration` es idempotente por nombre y **nunca borra**. Para
  eliminar un objeto hay que marcarlo `state: absent`.
- `infra.aap_configuration` lee las credenciales de un job template de la lista
  `credentials:`. La forma singular `credential:` se ignora sin aviso y el JT se
  crea sin credenciales.
- Un error en los ficheros de una torre hace fallar solo el nodo de esa torre;
  lo ya aplicado se mantiene.
- En una recuperación no hay que restaurar contraseñas: nunca estuvieron en AAP.
  Basta con repetir los pasos 4 a 7.

## Para producción

Este repositorio usa **un solo Key Vault y una sola source credential** para las
tres torres, y el CaC se ejecuta con una única cuenta con permisos sobre todas
las organizaciones. Con ese diseño, una torre puede referenciar un secreto de
otra si conoce su nombre. No puede crear secretos, pero sí usar los que existen.

Para aislar las torres:

- una source credential por torre, en la organización de la torre, con un
  service principal que solo pueda leer los secretos de esa torre (un Key Vault
  por torre, o RBAC por secreto);
- el CaC de cada torre ejecutado con una cuenta que solo administre su
  organización y solo pueda usar su source credential;
- revisión obligatoria por carpeta (CODEOWNERS) o un repositorio por torre.
