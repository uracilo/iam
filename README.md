### **Ejercicio IAM y S3**

## **Escenario**

La empresa ABC ha creado un bucket de S3 llamado `bucket-lab-iam-xidera-NOBRE`. Este bucket cuenta con dos carpetas principales:

1. **`/public/`**: contendrá archivos a los que ciertos usuarios podrán acceder con permisos de solo lectura.  
2. **`/private/`**: contendrá información interna. Únicamente un usuario administrador y un rol de respaldo tendrán acceso de lectura/escritura.  

Se requiere además que una aplicación que corre en **Amazon EC2** pueda **leer** archivos tanto en `/public/` como en `/private/`, pero que no pueda realizar operaciones de escritura ni borrado.

### **Requerimientos:**

1. **Grupo y política de solo lectura**  
   - Un grupo llamado `GrupoLectoresS3` donde se incluirán usuarios que solamente puedan **listar** y **leer** los objetos en la carpeta `/public/`.  
   - Estos usuarios **no** deben poder leer, subir ni borrar objetos en `/private/`.  

2. **Usuario administrador**  
   - Un usuario llamado `admin-s3` con **acceso total** (lectura, escritura y borrado) a todo el bucket.  
   - Este usuario debe poder ver tanto la carpeta `/public/` como la carpeta `/private/`, y realizar operaciones de subida y borrado de objetos en ambas.  

3. **Rol de IAM para EC2**  
   - Un rol para la instancia de EC2, llamado `EC2S3ReadOnlyRole`, que permita:  
     - **Leer** objetos en `/public/` y `/private/`.  
     - **Listar** el bucket completo.  
     - **No** se permitirá subir ni borrar objetos.  

4. **Requerimientos de seguridad adicionales**  
   - **Cifrado en reposo**: Asegúrate de que el bucket tenga **encryption** habilitado (SSE-S3 o SSE-KMS).  
   - **Bloqueo de acceso público**: Verifica que el bucket no sea público (usar las políticas de Bucket Policy o Access Block Settings).  
   - **Auditoría**: Habilita **CloudTrail** (si no está ya habilitado en la cuenta) para registrar las llamadas a la API de S3 e IAM, y así poder auditar los accesos.  

---

## **Pasos a Seguir**

A continuación se enumeran los pasos recomendados para llevar a cabo el ejercicio. Se incluye una estimación de tiempo que, sumada, te dará el total aproximado de **4 horas**.

### 1. Crear o verificar el bucket y la estructura de carpetas

1.1. **Crear el bucket** (si aún no existe) `bucket-lab-iam`.  
1.2. **Crear** las carpetas `public/` y `private/` dentro del bucket.  
1.3. **Subir archivos de prueba** en ambas carpetas.  
1.4. **Habilitar el cifrado en reposo** (SSE-S3 o SSE-KMS) en el bucket a nivel de configuración de S3.  


---

### 2. Bloquear el acceso público y configurar Bucket Policy (opcional)

2.1. En la consola de S3, revisa la sección **Block Public Access** y asegúrate de que **todas las opciones** de bloqueo de acceso público estén habilitadas.  
2.2. Verifica o edita la **Bucket Policy** (si existe) para cerciorarte de que no se permita acceso público ni permisos anónimos. (Si el bucket es nuevo y no requiere políticas adicionales, puedes omitir este paso.)  


---

### 3. Crear la **política de IAM** para “lectores de S3 (carpeta `/public/`)”

3.1. Ve a la consola de IAM y selecciona **Policies** (Políticas).  
3.2. Crea una política con nombre `LecturaS3PublicPolicy` que incluya:  
   - **Acciones**:  
     - `s3:ListBucket` sobre `arn:aws:s3:::bucket-lab-iam`.  
     - `s3:GetObject` sobre `arn:aws:s3:::bucket-lab-iam/public/*`.  
   - **No** permitir acceso a `bucket-lab-iam/private/*`.  
3.3. Guarda la política y verifica que aparezca en la lista de políticas personalizadas.  

Un ejemplo de la sección principal de la política podría ser:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::bucket-lab-iam"
    },
    {
      "Sid": "GetPublicObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::bucket-lab-iam/public/*"
    }
  ]
}
```


---

### 4. Crear el **grupo de IAM** y añadir usuarios de lectura

4.1. Crea un grupo llamado `GrupoLectoresS3`.  
4.2. Agrega la política `LecturaS3PublicPolicy` al grupo.  
4.3. Crea uno o dos usuarios, por ejemplo `usuario1` y `usuario2`.  
4.4. **Asigna** a ambos usuarios al `GrupoLectoresS3`.  


---

### 5. Crear y probar el usuario **administrador** de S3

5.1. Crea un usuario IAM llamado `admin-s3`.  
5.2. Asígnale una política de acceso total a S3, por ejemplo la administrada de AWS `AmazonS3FullAccess`, o crea una política personalizada que incluya `s3:*` para `arn:aws:s3:::bucket-lab-iam/*`.  
5.3. **Verificar**:  
   - Que `admin-s3` pueda listar y leer `/public/` y `/private/`.  
   - Que `admin-s3` pueda **subir** y **borrar** objetos en ambas carpetas.  


---

### 6. Crear el **rol de IAM** para EC2 con acceso de solo lectura

6.1. Ve a la consola de IAM → **Roles**.  
6.2. Crea un rol llamado `EC2S3ReadOnlyRole`.  
   - **Tipo de entidad** que asume el rol: **AWS service → EC2**.  
6.3. Asigna una política de solo lectura para todo el bucket. Puedes duplicar la política de “lectura” para `/public/` y ampliarla para `arn:aws:s3:::bucket-lab-iam/*`, o usar la política administrada `AmazonS3ReadOnlyAccess` de AWS (si quieres restringirlo solo a este bucket, crea una política personalizada).  
6.4. Configura tu instancia EC2 (nueva o existente) para que asuma el rol `EC2S3ReadOnlyRole` (en la sección **IAM Role** de la configuración de la instancia).  


---

### 7. **Habilitar CloudTrail** para auditoría (opcional pero recomendado)

7.1. Si no lo tienes habilitado, en la consola de CloudTrail, crea un **Trail** para registrar las llamadas a la API de S3 y de IAM a nivel de cuenta.  
7.2. Asocia un bucket de destino para los logs de CloudTrail (puede ser otro bucket diferente al de este ejercicio).  
7.3. Verifica que comience a registrar eventos de creación de usuarios, asociación de políticas, etc.  


---

### 8. **Pruebas de validación**

1. **Usuarios del grupo de lectura** (`usuario1` y `usuario2`):  
   - Con AWS CLI (o la consola web), **listar** el bucket. Debe funcionar.  
   - Intentar **descargar** un objeto de la carpeta `/public/`. Debe funcionar.  
   - Intentar **leer** un objeto de `/private/`. Debe **fallar** por falta de permisos.  
   - Intentar **subir** y/o **borrar** objetos en `/public/`. Debe fallar.  

2. **Usuario administrador** (`admin-s3`):  
   - **Listar** y **leer** todos los objetos.  
   - **Subir** un objeto a `/private/`. Debe funcionar.  
   - **Borrar** un objeto en `/public/`. Debe funcionar.  

3. **Instancia EC2 con rol `EC2S3ReadOnlyRole`**:  
   - Conéctate a la instancia EC2 (SSH o Session Manager).  
   - Ejecuta:  
     ```bash
     aws s3 ls s3://bucket-lab-iam
     aws s3 cp s3://bucket-lab-iam/private/ejemplo.txt .
     aws s3 cp s3://bucket-lab-iam/public/otro-archivo.txt .
     ```  
   - Verifica que **descargue** sin problemas.  
   - Intenta **subir** un archivo (ej. `aws s3 cp localfile.txt s3://bucket-lab-iam/private/`) y confirma que falle debido a permisos insuficientes.  

4. (Opcional) **Revisar CloudTrail**:  
   - Confirma que los eventos IAM y S3 estén siendo registrados.  


---

### 9. **Reporte final**

Para dar por concluido el ejercicio, se solicita un pequeño **reporte** que incluya:

1. **Capturas de pantalla o logs** de la consola o del AWS CLI demostrando:  
   - Los accesos exitosos/fallidos según cada usuario.  
   - Los accesos exitosos/fallidos desde la instancia EC2 con el rol.  
2. **Políticas en formato JSON**:  
   - `LecturaS3PublicPolicy`.  
   - La política de acceso completo utilizada (ya sea la administrada de AWS o una personalizada).  
   - La política asignada al rol `EC2S3ReadOnlyRole` (o, en su defecto, la referencia a la política administrada de AWS si usaste `AmazonS3ReadOnlyAccess`).  
3. **Configuración del rol** de IAM (`EC2S3ReadOnlyRole`) y evidencia de que se asoció correctamente a la instancia EC2.  
4. (Opcional) **Extracto de CloudTrail** donde se reflejen las operaciones realizadas.  

---
