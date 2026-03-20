# CLAUDE.md - Auto-cohort Plugin (local_cohortauto)

## Descripcion del Proyecto

Plugin local de Moodle que inscribe automaticamente usuarios en **cohortes** basandose en valores de campos de perfil usando plantillas **Mustache**. Reemplaza al plugin deprecado `auth_mcae`.

## Versiones Soportadas

- **Moodle 5.0.6** (Build: 20260216)
- **Moodle 5.1.1+** (Build: 20260109)
- **PHP 8.2+**

## Arquitectura del Plugin

```
local/cohortauto/
├── version.php              # Version del plugin y requisitos de Moodle
├── lib.php                  # Clase principal local_cohortauto_handler + helpers
├── settings.php             # Pagina de configuracion admin
├── view.php                 # Visor de miembros de cohortes
├── convert.php              # Herramienta de migracion/conversion de cohortes
├── styles.css               # Estilos CSS para paginas admin
├── classes/
│   ├── observer.php         # Observer de eventos user_created / user_updated
│   └── privacy/
│       └── provider.php     # Privacy API (null_provider)
├── cli/
│   ├── sync_user.php        # CLI: sincronizar cohortes de un usuario
│   ├── sync_users.php       # CLI: sincronizar cohortes de todos los usuarios
│   └── auth_mcae_convert.php # CLI: migrar desde auth_mcae
├── db/
│   └── events.php           # Registro de observers de eventos
└── lang/
    └── en/
        └── local_cohortauto.php  # Cadenas de idioma (ingles)
```

## Flujo de Funcionamiento

1. Un usuario se crea o actualiza (evento `user_created` / `user_updated`)
2. El observer (`classes/observer.php`) captura el evento
3. Se instancia `local_cohortauto_handler` (en `lib.php`)
4. Se carga el perfil del usuario + campos personalizados
5. Se aplican las plantillas Mustache para generar nombres de cohortes
6. Se aplican reemplazos de texto (`replace_arr`)
7. Se crea la cohorte si no existe y se inscribe al usuario
8. Opcionalmente se desinscribe de cohortes que ya no aplican (`enableunenrol`)

## Archivos Clave y su Funcion

### lib.php (Logica principal)
- `cohortauto_prepare_profile_data()` - Limpia y prepara datos del perfil de usuario
- `cohortauto_print_profile_data()` - Genera ayuda con variables disponibles para settings
- `local_cohortauto_handler` - Clase principal:
  - `__construct()` - Carga Mustache (con fallbacks para 5.0 y 5.1) y config
  - `process_config($config)` - Valida y guarda configuracion
  - `user_profile_hook(&$user)` - Procesa la inscripcion automatica en cohortes

### settings.php (Configuracion)
Opciones en "Administracion del sitio > Plugins > Plugins locales":
- `mainrule_fld` - Plantilla Mustache principal (una por linea)
- `delim` - Delimitador de lineas (CR+LF, CR, LF)
- `secondrule_fld` - Texto cuando el campo esta vacio
- `replace_arr` - Reemplazos (formato: `viejo|nuevo`)
- `donttouchusers` - Usuarios a ignorar (separados por coma)
- `enableunenrol` - Auto-desuscripcion de cohortes gestionadas

### db/events.php (Eventos)
Registra dos observers:
- `\core\event\user_created` → `\local_cohortauto\observer::user_created`
- `\core\event\user_updated` → `\local_cohortauto\observer::user_updated`

## Sistema de Plantillas Mustache

### Variables disponibles
- Campos estandar: `{{ username }}`, `{{ firstname }}`, `{{ lastname }}`, `{{ idnumber }}`, etc.
- Email: `{{ email.full }}`, `{{ email.username }}`, `{{ email.domain }}`, `{{ email.rootdomain }}`
- Campos personalizados: `{{ profile.campo }}` o `{{ profile_field_campo }}`

### Funcion especial %split
```
%split(campo|delimitador)
```
Divide un campo en multiples valores y crea cohortes separadas.

### Ejemplo
```
Plantilla: {{ profile_field_classcode }} - {{ profile_field_status }}s
Reemplazo: none - admins|Administradores
```
Un usuario con classcode="Y19A" y status="student" → cohorte "Y19A - students"

## Tablas de Moodle Utilizadas (sin tablas propias)

- `{cohort}` - Registros de cohortes (`component = 'local_cohortauto'` para las gestionadas)
- `{cohort_members}` - Membresias usuario-cohorte
- `{user}` - Datos de perfil
- `{user_info_data}` - Campos personalizados de perfil

## Compatibilidad Moodle 5.0 / 5.1

### Diferencias manejadas
- **Carga de Mustache**: Se busca en multiples paths (`lib/mustache/` y `vendor/mustache/`) con fallback
- **`str_contains()`**: Requiere PHP 8.0+ (garantizado con PHP 8.2+)
- **Context API**: Usa `context_system::instance()` (alias compatible en ambas versiones)
- **Cohort API**: `cohort_add_cohort()`, `cohort_add_member()`, `cohort_remove_member()` son estables
- **Privacy API**: `null_provider` interface es estable en ambas versiones

### Al modificar el plugin
- Siempre verificar que las funciones de Moodle core usadas existan en AMBAS versiones (5.0 y 5.1)
- No usar funciones deprecadas exclusivas de una version
- Mantener `$plugin->requires = 2024100700` en version.php para soportar Moodle 4.5+
- Probar en ambas instancias de Moodle antes de desplegar

## Comandos CLI

```bash
# Sincronizar cohortes de un usuario especifico
php local/cohortauto/cli/sync_user.php

# Sincronizar cohortes de todos los usuarios
php local/cohortauto/cli/sync_users.php

# Migrar cohortes de auth_mcae a local_cohortauto
php local/cohortauto/cli/auth_mcae_convert.php
```

## Convenciones de Codigo

- **Paquete**: `local_cohortauto`
- **Componente**: `local_cohortauto` (usado en campo `component` de tabla cohort)
- **Namespace**: `local_cohortauto\` para clases en `classes/`
- **Licencia**: GNU GPL v3+
- **PHP**: Compatible con PHP 8.2+
- **Estilo**: Seguir los coding standards de Moodle (https://moodledev.io/general/development/policies/codingstyle)

## Instalacion

1. Copiar la carpeta del plugin a `{moodle_root}/local/cohortauto/`
2. Ir a "Administracion del sitio > Notificaciones" para ejecutar la instalacion
3. Configurar en "Administracion del sitio > Plugins > Plugins locales > Auto-cohort plugin"

## Migracion desde auth_mcae

- Via web: "Administracion del sitio > Usuarios > Cuentas > CohortAuto conversion operations"
- Via CLI: `php local/cohortauto/cli/auth_mcae_convert.php`
- Cambia el campo `component` de `auth_mcae` a `local_cohortauto` en la tabla cohort

## Notas Importantes

- El plugin NO crea tablas propias en la base de datos
- Las cohortes gestionadas tienen `component = 'local_cohortauto'` y NO se pueden editar manualmente
- El campo `enableunenrol` controla si se remueven usuarios de cohortes que ya no les corresponden
- Los usuarios en la lista `donttouchusers` y los invitados son ignorados completamente
- Siempre hacer backup de la base de datos antes de operaciones de conversion
