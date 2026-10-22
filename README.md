# Fedora Server: Repositorio seguro de tareas mediante ACL

Proyecto académico de administración de sistemas Linux desarrollado para la asignatura **Gestión de Sistemas Operativos II** de la Licenciatura en Ciberseguridad de la Universidad Tecnológica de Panamá.

El proyecto implementa un repositorio compartido de tareas en Fedora Server 44, aplicando gestión de usuarios y grupos, permisos tradicionales de Linux, listas de control de acceso (ACL) y protección contra la eliminación de archivos ajenos mediante el sticky bit.

## 1. Objetivo

Diseñar e implementar un mecanismo de entrega de archivos para un entorno educativo en el que varios estudiantes utilizan un mismo directorio, pero no deben poder consultar ni eliminar las tareas de sus compañeros.

La solución contempla los siguientes requisitos:

- Todos los estudiantes pueden acceder al directorio común y depositar sus propias tareas.
- Los estudiantes no pueden listar el contenido del directorio.
- Las tareas creadas heredan permisos ACL que restringen el acceso al grupo `estudiantes`.
- El sticky bit impide que un estudiante elimine o renombre las entregas de otros estudiantes.
- La coordinadora académica puede listar el repositorio y leer las entregas.
- Los archivos `.bashrc` y `.bash_profile` de los estudiantes quedan protegidos mediante permisos y el atributo inmutable.
- Se realizan pruebas con diferentes usuarios para comprobar el comportamiento de los permisos y las ACL.

## 2. Caso organizacional

**Organización:** Instituto de Idiomas UTP  
**Sector:** Educación  
**Sistema operativo:** Fedora Server 44  
**Entorno de virtualización:** VMware Workstation Pro 25H2u1

El instituto necesita un repositorio centralizado para recibir las tareas de sus estudiantes. La coordinadora debe poder revisar todas las entregas, mientras que los estudiantes no deben tener acceso de lectura a los archivos de sus compañeros, ni siquiera cuando conozcan el nombre de un archivo.

Para resolver el caso se utiliza el directorio `/srv/tareas`, el grupo común `estudiantes` y las ACL de Linux.

## 3. Diseño de la solución

### Usuarios y grupo

| Usuario o grupo | Función |
|---|---|
| `estudiante1` | Usuario de prueba que deposita una tarea. |
| `estudiante2` | Segundo usuario de prueba para verificar el aislamiento entre estudiantes. |
| `prof_lucia` | Coordinadora académica y propietaria del directorio de entregas. |
| `estudiantes` | Grupo común al que pertenecen los estudiantes para aplicar permisos compartidos. |

### Directorio de entregas

La ubicación elegida es:

`/srv/tareas`

Este directorio concentra las entregas y mantiene los archivos separados de las carpetas personales de cada usuario.

### Permisos tradicionales y ACL

La solución combina varias configuraciones:

- **Permisos tradicionales `700`:** establecen permisos completos para la propietaria del directorio y ningún permiso tradicional para el grupo propietario ni para otros usuarios.
- **ACL de acceso `g:estudiantes:wx`:** permite al grupo acceder al directorio y depositar archivos sin conceder el permiso de lectura necesario para listar su contenido.
- **Default ACL `g:estudiantes:---`:** establece una entrada de herencia que deniega permisos al grupo `estudiantes` en los archivos nuevos.
- **Default ACL `u:prof_lucia:rw`:** concede a la coordinadora lectura y escritura sobre los archivos que heredan esta ACL.
- **Sticky bit:** impide que un estudiante elimine o renombre archivos de otros usuarios dentro del directorio compartido. La propietaria del directorio y el administrador conservan la capacidad de gestionar las entregas.
- **Protección de archivos Bash:** los archivos `.bashrc` y `.bash_profile` cambian de propietario a `prof_lucia`, se establecen con modo `644` y reciben el atributo inmutable mediante `chattr +i`.

### ¿Por qué utilizar ACL y sticky bit?

Los permisos `wx` permiten depositar archivos sin conceder el permiso de listado. Sin embargo, la escritura sobre un directorio permite realizar operaciones sobre sus entradas, como eliminar archivos, cuando no hay restricciones adicionales.

El sticky bit añade esa restricción: un estudiante puede gestionar sus propios archivos, pero no eliminar ni renombrar las entregas de otros estudiantes únicamente por tener permiso de escritura sobre el directorio.

Las ACL predeterminadas, por su parte, restringen los permisos heredados por los nuevos archivos. Esto permite limitar la lectura de las tareas incluso si un estudiante cambia su máscara de creación mediante `umask`.

## 4. Implementación

Los siguientes bloques contienen los comandos de configuración del proyecto, incorporando únicamente las modificaciones relacionadas con el sticky bit, la protección inmutable de los archivos Bash y la eliminación de la referencia al usuario inexistente.

Las operaciones administrativas deben ejecutarse con privilegios suficientes. Los comandos se muestran desde la perspectiva de una sesión administrativa en Fedora Server.

### Paso 1. Crear el grupo y los usuarios

```bash
groupadd estudiantes

# Crear los usuarios con sus carpetas personales
useradd -m -g estudiantes estudiante1
useradd -m -g estudiantes estudiante2
useradd -m prof_lucia

sudo passwd estudiante1
sudo passwd estudiante2
```

Este bloque crea el grupo `estudiantes`, dos cuentas de estudiantes y la cuenta de la coordinadora. A continuación, permite establecer las contraseñas de los dos estudiantes.

### Paso 2. Crear el directorio y configurar sus permisos

```bash
# Crear el directorio raíz para el repositorio digital de tareas
mkdir -p /srv/tareas

# Asignar a la coordinadora como propietaria exclusiva y al grupo estudiantes
chown prof_lucia:estudiantes /srv/tareas

# Limpiar permisos tradicionales del directorio base (Solo dueña tiene control total)
chmod 700 /srv/tareas


# Permitir a los estudiantes ingresar (x) y escribir/depositar (w) en la carpeta
setfacl -m g:estudiantes:wx /srv/tareas

# CONFIGURACIÓN CRÍTICA: Definir la directiva de herencia (Default ACL)
# Esto obliga a que CUALQUIER archivo nuevo bloquee al grupo estudiantes y dé acceso a la prof.
setfacl -d -m g:estudiantes:--- /srv/tareas
setfacl -d -m u:prof_lucia:rw /srv/tareas

# Impedir que los estudiantes eliminen o renombren archivos ajenos
chmod +t /srv/tareas
```

El directorio se crea con la propietaria `prof_lucia` y el grupo `estudiantes`. Las ACL permiten a los estudiantes acceder al directorio y escribir en él sin poder listar su contenido.

Las Default ACL establecen permisos heredados para los archivos nuevos. El comando `chmod +t` activa el sticky bit para restringir la eliminación y el renombrado de archivos ajenos.

Se recomienda comprobar el resultado con:

```bash
ls -ld /srv/tareas
getfacl /srv/tareas
```

En el resultado de `ls -ld`, el indicador `t` o `T` muestra que el sticky bit está activo. La forma exacta de visualizar los permisos también depende de las ACL presentes.

### Paso 3. Cambiar la propiedad y proteger los archivos de sesión Bash

```bash
# Cambiar la propiedad de los perfiles para impedir su edición directa por el estudiante
chown prof_lucia:prof_lucia /home/estudiante1/.bashrc /home/estudiante1/.bash_profile
chown prof_lucia:prof_lucia /home/estudiante2/.bashrc /home/estudiante2/.bash_profile

# Asegurar que queden estrictamente en modo lectura para el alumno
chmod 644 /home/estudiante1/.bashrc /home/estudiante1/.bash_profile
chmod 644 /home/estudiante2/.bashrc /home/estudiante2/.bash_profile

# Marcar los archivos como inmutables para impedir su modificación, eliminación o sustitución
chattr +i /home/estudiante1/.bashrc /home/estudiante1/.bash_profile
chattr +i /home/estudiante2/.bashrc /home/estudiante2/.bash_profile
```

Cambiar la propiedad y aplicar el modo `644` impide que los estudiantes escriban directamente en estos archivos mediante los permisos tradicionales. Sin embargo, como controlan sus directorios personales, esas medidas por sí solas no impiden necesariamente que eliminen o sustituyan las entradas.

El atributo inmutable `i` refuerza la protección: mientras permanezca activo y el sistema de archivos lo admita, el archivo no puede modificarse, eliminarse ni renombrarse. Esta protección también impide que la coordinadora lo edite directamente hasta retirar temporalmente el atributo.

**Comprobación del atributo inmutable:**

```bash
lsattr /home/estudiante1/.bashrc /home/estudiante1/.bash_profile
lsattr /home/estudiante2/.bashrc /home/estudiante2/.bash_profile
```

El resultado debe mostrar la letra `i` en los atributos de los cuatro archivos. Si `chattr +i` falla, hay que comprobar que el sistema de archivos admite el atributo inmutable antes de dar por aplicada la protección.

### Administración posterior de los archivos Bash

Si la coordinadora necesita modificar uno de estos archivos, el administrador debe retirar temporalmente el atributo inmutable, realizar los cambios y volver a activarlo:

```bash
chattr -i /home/estudiante1/.bashrc /home/estudiante1/.bash_profile
chattr -i /home/estudiante2/.bashrc /home/estudiante2/.bash_profile

# Realizar aquí los cambios administrativos necesarios.

chattr +i /home/estudiante1/.bashrc /home/estudiante1/.bash_profile
chattr +i /home/estudiante2/.bashrc /home/estudiante2/.bash_profile
```

Estos comandos son para mantenimiento administrativo; no forman parte de la sesión habitual del estudiante.

La protección de los archivos Bash no impide que un usuario ejecute `umask 000` directamente en su sesión. Por eso las ACL predeterminadas del directorio de entregas siguen siendo necesarias: no se debe depender exclusivamente de los archivos de configuración de Bash para proteger las tareas.

## 5. Validación como estudiante

### Paso 1. Iniciar sesión

```bash
su - estudiante1
```

### Paso 2. Intentar modificar el entorno

```bash
# 1. Intentar alterar el entorno (Mostrará error)
nano ~/.bashrc
# El archivo no debe poder guardarse debido a los permisos y al atributo inmutable.
```

La prueba intenta modificar `.bashrc`. Aunque el editor permita abrir el archivo o modificar temporalmente su contenido en memoria, el estudiante no debe poder guardar los cambios, eliminarlo ni sustituirlo mientras el atributo inmutable esté activo.

El mensaje exacto puede variar según la versión del editor y el sistema.

### Paso 3. Modificar la máscara de creación

```bash
# 2. Intentar vulnerar la máscara con umask 000
umask 000
```

El comando modifica la máscara de creación de archivos de la sesión. Se utiliza para comprobar que las ACL predeterminadas siguen restringiendo el acceso de lectura del grupo en las tareas nuevas.

### Paso 4. Crear una tarea

```bash
# 3. Crear el archivo de la tarea dentro del buzón ciego
echo "Contenido confidencial de E1" > /srv/tareas/tarea_estudiante1.txt
```

El comando deposita una tarea en el directorio compartido. El usuario puede crear el archivo porque dispone de los permisos de acceso necesarios sobre el directorio.

### Paso 5. Intentar listar el directorio

```bash
# 4. Intentar listar la carpeta para ver tareas ajenas (Mostrará error)
ls -la /srv/tareas/
# Salida esperada: ls: cannot open directory '/srv/tareas/': Permission denied
```

El resultado esperado es un acceso denegado al intentar listar las entregas. Esta prueba comprueba el comportamiento de los permisos del directorio.

```bash
exit
```

El comando termina la sesión de `estudiante1` y devuelve al usuario a la sesión anterior.

## 6. Validación del aislamiento entre estudiantes

### Paso 1. Iniciar sesión con el segundo estudiante

```bash
su - estudiante2
```

### Paso 2. Intentar leer la tarea del primer estudiante

```bash
cat /srv/tareas/tarea_estudiante1.txt
```

El acceso debe ser denegado porque la ACL heredada no concede permisos de lectura al grupo `estudiantes` sobre el archivo.

### Paso 3. Intentar eliminar la tarea ajena

```bash
rm /srv/tareas/tarea_estudiante1.txt
```

El sistema debe impedir que `estudiante2` elimine el archivo de `estudiante1` debido al sticky bit del directorio compartido.

La denegación de lectura y la denegación de eliminación comprueban dos controles diferentes: las ACL restringen el acceso al contenido y el sticky bit limita la gestión de las entradas del directorio.

```bash
exit
```

Estas dos pruebas deben ejecutarse después de crear realmente la tarea de `estudiante1` y activar el sticky bit. Los resultados no se deben considerar comprobados hasta ejecutarlas en el entorno.

## 7. Validación como coordinadora

### Paso 1. Iniciar sesión

```bash
su - prof_lucia
```

### Paso 2. Listar las entregas

```bash
# 1. Listar el repositorio completo de entregas
ls -la /srv/tareas/
# Salida esperada: Mostrará todos los archivos creados por los alumnos
```

La coordinadora debe poder listar el directorio y consultar las entradas creadas por los estudiantes.

### Paso 3. Consultar las ACL de una tarea

```bash
# 2. Verificar los metadatos y permisos reales heredados por las ACLs
getfacl /srv/tareas/tarea_estudiante1.txt
# Salida esperada:
# # file: /srv/tareas/tarea_estudiante1.txt
# # owner: estudiante1
# # group: estudiantes
# user:prof_lucia:rw-     <-- Acceso de lectura garantizado para la coordinadora
# group:estudiantes:---   <-- Alumnos bloqueados a nivel de Kernel (Ignoró el umask 000)
```

La consulta permite inspeccionar la propiedad y las entradas ACL del archivo. La entrada para `prof_lucia` debe permitir la lectura del contenido, mientras que la entrada del grupo `estudiantes` no debe conceder permisos.

La salida real de `getfacl` puede incluir otras entradas, como las ACL de propietario, grupo base, máscara y otros usuarios.

### Paso 4. Leer las tareas

```bash
# 3. Leer las tareas de forma directa
cat /srv/tareas/tarea_estudiante1.txt
cat /srv/tareas/tarea_estudiante2.txt
```

La coordinadora utiliza `cat` para consultar directamente los archivos de tareas. El bloque presupone que ambos archivos existen.

## 8. Resultados documentados

El informe académico recoge evidencias de pruebas realizadas con estudiantes y con la coordinadora.

| Prueba | Resultado esperado |
|---|---|
| Modificación de `.bashrc` por el estudiante | No puede guardar cambios mientras el archivo mantenga los permisos y el atributo inmutable configurados. |
| Ejecución de `umask 000` | El comando se ejecuta en la sesión del usuario, pero no debe anular las ACL predeterminadas. |
| Creación de una tarea en `/srv/tareas` | El estudiante puede depositar su archivo. |
| Listado del directorio por un estudiante | Se deniega el acceso al listado. |
| Lectura de una tarea ajena por un estudiante | Se deniega si las ACL efectivas del archivo no conceden lectura al estudiante. |
| Eliminación de una tarea ajena | Se deniega por el sticky bit, mientras el usuario no sea propietario del archivo ni del directorio y no tenga privilegios administrativos. |
| Listado del directorio por la coordinadora | Puede consultar las entregas. |
| Lectura de las tareas por la coordinadora | Puede leer las entregas si las ACL efectivas le conceden permisos. |
| Consulta de ACL con `getfacl` | Permite examinar las entradas de acceso y de herencia del archivo. |
| Consulta con `lsattr` | Permite comprobar si el atributo inmutable está activo en los archivos Bash. |

Los resultados se deben confirmar mediante pruebas en el servidor. La documentación de una prueba no sustituye su ejecución.

## 9. Consideraciones técnicas y limitaciones

### ACL predeterminadas y `umask`

Una Default ACL se hereda al crear archivos dentro de un directorio. Cuando existe una Default ACL, el comportamiento de los permisos heredados difiere del caso en que solo se utiliza `umask`.

En este proyecto se configura una entrada predeterminada `g:estudiantes:---`, que busca impedir que los archivos creados otorguen permisos de lectura o escritura al grupo de estudiantes. Por ello, ejecutar `umask 000` no debería conceder al grupo los permisos que la ACL heredada le deniega.

El resultado efectivo debe comprobarse mediante `getfacl` sobre cada archivo.

### Sticky bit

El sticky bit impide que un usuario sin privilegios elimine o renombre archivos ajenos dentro del directorio compartido cuando no es propietario del archivo ni del directorio.

En esta implementación, `prof_lucia` es la propietaria de `/srv/tareas`, por lo que conserva la capacidad de gestionar las entregas. Los estudiantes pueden depositar sus propios archivos, pero no deberían poder borrar o renombrar las tareas de los demás.

El sticky bit no reemplaza las ACL: ambas configuraciones resuelven aspectos diferentes del control de acceso.

### Archivos de configuración Bash

El cambio de propietario y el modo `644` restringen la escritura directa por parte del estudiante. El atributo inmutable añade protección frente a modificación, eliminación y renombrado del archivo.

La coordinadora no podrá editar directamente estos archivos mientras el atributo inmutable esté activo. Para modificarlos, debe retirarlo temporalmente y volver a aplicarlo después.

El atributo `i` requiere soporte del sistema de archivos y privilegios administrativos para activarlo o retirarlo. La salida de `lsattr` debe revisarse para confirmar que la protección quedó aplicada.

### Alcance de la solución

La implementación se centra en la administración de usuarios, grupos, permisos tradicionales, ACL, sticky bit y atributos de archivos en Fedora Server.

No representa un sistema completo de gestión de entregas con auditoría, historial de versiones, protección contra cambios de los propietarios de archivos o mecanismos de recuperación. El administrador y la propietaria del directorio conservan privilegios de gestión sobre las entregas.

## 10. Aprendizajes

El proyecto permitió integrar varios conceptos de administración de Linux:

- Creación y gestión de usuarios y grupos.
- Organización de directorios para un caso organizacional.
- Diferencias entre los permisos de archivos y directorios.
- Aplicación de permisos tradicionales mediante `chmod` y `chown`.
- Uso de ACL de acceso y ACL predeterminadas mediante `setfacl`.
- Inspección de permisos mediante `getfacl`.
- Restricción de eliminación y renombrado mediante el sticky bit.
- Protección de archivos de configuración mediante el atributo inmutable.
- Pruebas de control de acceso usando distintas identidades de usuario.
- Evaluación de las limitaciones de una solución de permisos en un entorno compartido.

La experiencia muestra que el control de acceso en Linux requiere combinar el diseño de permisos con pruebas prácticas y análisis de las acciones que los usuarios pueden realizar.

## 11. Referencias

- Cyberciti.biz. (s. f.). *What is umask and how to setup default umask under Linux?* https://www.cyberciti.biz/tips/understanding-linux-unix-umask-value-usage.html
- GoLinuxCloud. (s. f.). *Linux ACL: setfacl, getfacl, and permission workflow on ext4 and xfs.* https://www.golinuxcloud.com/linux-acl-examples/
- Linux man-pages. (s. f.). *chmod(1) — change file mode bits.* https://man7.org/linux/man-pages/man1/chmod.1.html
- Linux man-pages. (s. f.). *chattr(1) — change file attributes on a Linux file system.* https://man7.org/linux/man-pages/man1/chattr.1.html
