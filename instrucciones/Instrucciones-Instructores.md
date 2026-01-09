# 📘 INSTRUCTOR – Orden Recomendado de Edición

Sigue estos pasos en orden para configurar correctamente el curso.  
**Nota:** Solo modifica lo indicado en los comentarios de los archivos.

---

## 🔹 Pasos de Configuración

1. **Instalación inicial**  
   - Realizar las instrucciones del archivo **`Instalacion-Instructores.md`**.

2. **Configurar cursos**  
   - Editar el archivo **`_data/courses.yml`** (revisar comentarios dentro del archivo).

3. **Configurar resultados de tareas**  
   - Editar el archivo **`_data/task-results.yml`** (revisar comentarios dentro del archivo).

4. **Ajustar layout del curso**  
   - En **`_layouts/course.html`**, editar la **línea 13** con el mismo `ID_CURSO` que aparece en la **línea 2** de **`_data/courses.yml`**.

5. **Archivo principal del curso**  
   - El archivo en el directorio **`curso/nombre-curso.md`** debe tener el mismo nombre que en el `ID_CURSO` **`_data/courses.yml`**.

6. **Contenido del curso**  
   - Editar el archivo **`curso/nombre-curso.md`**.  
   - ⚠️ **Modificar únicamente lo que tiene comentarios.**

7. **Laboratorios (Labs)**  
   - La carpeta **`labs/`** contiene la relación de laboratorios por capítulo.  
     - **`labs/lab#`** → agrega cuantos sean necesarios.  
     - **`labs/lab#/img`** → contiene las imágenes necesarias por laboratorio.  
     - **`labs/lab#/lab#.md`** → detalle de cada práctica y pasos (revisar instrucciones dentro).

      ## Crear nuevos labs

      Desde la raíz del proyecto:

      ```bash
      ./scripts/create_labs.sh 5
      ```

      Esto creará:

      - labs/lab1/lab1.md + labs/lab1/img/
      - labs/lab2/lab2.md + labs/lab2/img/
      - ...
      - labs/lab5/lab5.md + labs/lab5/img/


8. **Archivo de configuración**  
   - En **`_config.yaml`**, ajustar solo los valores con comentarios.  
   - ⚠️ No modificar el resto.

9. **Archivo README**  
   - Editar el archivo **`README.md`** para escribir el **nombre del curso**.

10. **Revisión final**  
    - Revisar el archivo **`GitHubPush-Instructores.md`**.

---

## 🚫 INSTRUCTORES – Archivos y Directorios que **NO se deben modificar**

- `_includes/`
- `_layouts/`
- `.github/`
- `assets/`
- `.gitignore`
- `Gemfile`
- `Gemfile.lock`

---
