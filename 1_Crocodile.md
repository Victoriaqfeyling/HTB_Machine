# 🐊 Hack The Box - Crocodile Writeup

## 📌 Información general

* **Nombre:** Crocodile
* **Plataforma:** Hack The Box - Starting Point
* **Dificultad:** Muy fácil / Fácil
* **Sistema objetivo:** Linux
* **Temáticas principales:** Enumeración, FTP anónimo, exposición de credenciales, acceso web

---

## 🎯 Objetivo del laboratorio

El objetivo de esta máquina es aprender a:

* identificar servicios expuestos mediante reconocimiento inicial,
* detectar configuraciones inseguras en FTP,
* descargar archivos sensibles desde un servidor accesible de forma anónima,
* reutilizar credenciales encontradas para acceder a una aplicación web,
* obtener la flag de usuario.

Esta máquina es ideal para reforzar un concepto muy importante en pentesting:

> una mala configuración aparentemente simple puede derivar en un compromiso completo del sistema o de la aplicación.

---
Para iniciar el laboratorio de HTB nos conectamos a la vpn y realizamos un ```ip a``` para ver si responde. Seguidamente, copiamos la IP de la máquina a resolver y mediante ```ping -c 1 <IP>``` ************

<img width="1278" height="256" alt="image" src="https://github.com/user-attachments/assets/4fda85da-01ba-4aeb-96fd-dee9af70259d" />




# 1. Reconocimiento inicial

Como primer paso, realizamos un escaneo con **Nmap** para identificar puertos abiertos, servicios y versiones.

```bash
nmap -sC -sV -p- -T4 <IP>
```

## Explicación del comando

* `-sC` → ejecuta scripts NSE básicos por defecto.
* `-sV` → detecta versiones de servicios.

---

## Resultado esperado

<img width="1294" height="639" alt="image" src="https://github.com/user-attachments/assets/749f8fbf-87f5-4700-ba14-94b6ab7112cb" />



```bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
80/tcp open  http    Apache httpd 2.4.41
```

---

## Análisis inicial

A partir de este resultado ya podemos extraer varias conclusiones:

1. **El puerto 21/TCP está abierto**, por lo que hay un servicio FTP expuesto.
2. **El puerto 80/TCP está abierto**, por lo que probablemente existe una aplicación o sitio web accesible desde el navegador.
3. La presencia simultánea de **FTP + HTTP** suele ser interesante, porque muchas veces FTP expone archivos que luego pueden reutilizarse contra la aplicación web.

En este punto ya tenemos dos superficies de ataque claras:

* **FTP** → posible acceso anónimo o fuga de archivos.
* **HTTP** → posible panel de login o aplicación web vulnerable.

---

# 2. Enumeración del servicio FTP

Dado que FTP suele ser un vector simple pero muy útil en laboratorios de iniciación, el siguiente paso lógico es intentar una conexión manual.

```bash
ftp <IP>
```

Cuando el servidor solicite credenciales, probamos el acceso anónimo:

```text
Name: anonymous
Password: anonymous
```

> En muchos servidores FTP antiguos o mal configurados, el usuario `anonymous` está habilitado y permite descargar archivos sin autenticación real.

---

## Resultado esperado

```text
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

---

<img width="747" height="220" alt="image" src="https://github.com/user-attachments/assets/cd5f1eec-3b23-40d6-92e7-557d39775ec0" />


## Análisis

Si el login anónimo funciona, estamos ante una **mala configuración de seguridad**.

Esto implica que un atacante sin credenciales válidas puede:

* listar archivos,
* descargar contenido,
* recopilar información sensible,
* encontrar credenciales reutilizables.

---

# 3. Enumeración de archivos dentro de FTP

Una vez autenticados, listamos el contenido del directorio remoto:

```bash
ls
```
<img width="1786" height="210" alt="image" src="https://github.com/user-attachments/assets/a168b474-c118-4473-acef-f0156d4310a1" />

## Resultado esperado

```text
allowed.userlist
allowed.userlist.passwd
```

---

## Análisis de los nombres de archivo

Estos nombres ya son una alerta por sí mismos:

* `allowed.userlist` sugiere una lista de usuarios permitidos.
* `allowed.userlist.passwd` sugiere contraseñas asociadas o algún archivo relacionado con credenciales.

En pentesting, cuando encontramos archivos con nombres relacionados con:

* `user`,
* `passwd`,
* `creds`,
* `backup`,
* `config`,

siempre deben tratarse como **objetivos prioritarios**.

---

# 4. Descarga de archivos sensibles

Descargamos ambos archivos a nuestra máquina atacante:

```bash
get allowed.userlist
get allowed.userlist.passwd
```
<img width="1182" height="81" alt="image" src="https://github.com/user-attachments/assets/400cc261-641f-431d-acbd-9a213129000f" />

<img width="1890" height="229" alt="image" src="https://github.com/user-attachments/assets/d438bea3-3f78-473b-9dba-0a688f2503f3" />

<img width="1920" height="265" alt="image" src="https://github.com/user-attachments/assets/c62c0294-5286-4f85-8a51-5f0bbbfc3e4c" />


También podríamos usar `mget *` si quisiéramos bajar varios archivos a la vez, pero en este caso alcanza con descargar los dos archivos relevantes.

---

## Verificación local

Una vez descargados, confirmamos que estén presentes:

```bash
ls -la
```
<img width="1491" height="285" alt="image" src="https://github.com/user-attachments/assets/16270250-4311-479e-a9f2-9fe678d17d6a" />

Luego leemos su contenido:

```bash
cat allowed.userlist
cat allowed.userlist.passwd
```

<img width="687" height="196" alt="image" src="https://github.com/user-attachments/assets/d3b73e3e-77c1-4b17-a7d3-ee6ca9db17c9" />

<img width="637" height="162" alt="image" src="https://github.com/user-attachments/assets/5d2be688-33ea-4311-9619-ec4db04c3462" />


---

## Qué buscamos acá

En esta etapa buscamos:

* nombres de usuario,
* contraseñas en texto plano,
* patrones de naming,
* relaciones uno a uno entre usuarios y passwords.

En Crocodile, el punto clave es que los archivos exponen información de autenticación que puede reutilizarse en otro servicio.

---

# 5. Hipótesis de ataque

Con FTP ya enumerado, formulamos una hipótesis muy razonable:

> las credenciales encontradas podrían servir para autenticarse en la aplicación web del puerto 80.

Este tipo de encadenamiento es extremadamente común:

* un servicio expone información,
* esa información permite comprometer otro servicio,
* el segundo servicio entrega acceso más sensible.

---

# 6. Enumeración web

Abrimos el sitio en el navegador:

```text
http://<IP>
```

## Qué observar

Al cargar la web, debemos prestar atención a:

* si se trata de una página estática o dinámica,
* si existe un formulario de login,
* si hay rutas interesantes,
* si hay pistas sobre usuarios o paneles administrativos.

En esta máquina, lo importante es detectar que existe una **interfaz de autenticación web**.

---

# 7. Prueba de credenciales encontradas

Con los usuarios y contraseñas obtenidos desde FTP, comenzamos a probar combinaciones en el login web.

## Enfoque recomendado

1. Identificar usuarios válidos en `allowed.userlist`.
2. Identificar contraseñas candidatas en `allowed.userlist.passwd`.
3. Probar combinaciones lógicas manualmente.

Ejemplo conceptual:

```text
Usuario: admin
Password: <valor obtenido en el archivo>
```

> En lugar de pensar solo en “fuerza bruta”, en este caso el valor real del ejercicio está en la **reutilización de credenciales expuestas**.

---

## Resultado esperado

Después de probar la combinación correcta, el login es exitoso y obtenemos acceso a una sección restringida de la aplicación.

---

# 8. Acceso al panel y obtención de la flag

Una vez autenticados correctamente, ingresamos al panel o sección interna del sitio.

En esta instancia, el laboratorio normalmente revela la flag directamente dentro de la interfaz.

### Acción final

* Buscar la flag en la pantalla.
* Copiarla.
* Validarla en Hack The Box.

---

# 9. Resumen de la cadena de compromiso

La explotación de Crocodile puede resumirse así:

1. Escaneo con Nmap.
2. Identificación de FTP y HTTP.
3. Acceso anónimo a FTP.
4. Descarga de archivos sensibles.
5. Obtención de usuarios y contraseñas.
6. Reutilización de credenciales en el panel web.
7. Acceso exitoso.
8. Obtención de la flag.

---

# 10. Vulnerabilidades identificadas

## 10.1. FTP anónimo habilitado

El servidor FTP permite acceso con el usuario `anonymous`, lo cual expone información a usuarios no autenticados.

### Riesgo

* filtración de archivos,
* enumeración de estructura interna,
* acceso a información sensible.

---

## 10.2. Exposición de credenciales en texto plano

Los archivos descargables contienen información sensible reutilizable.

### Riesgo

* compromiso de cuentas,
* movimiento lateral entre servicios,
* acceso no autorizado.

---

## 10.3. Reutilización de credenciales entre servicios

La misma información obtenida desde FTP permite comprometer el acceso web.

### Riesgo

* escalamiento de impacto entre componentes,
* ruptura del aislamiento lógico entre servicios.

---

# 11. Impacto de seguridad

Si este escenario existiera en un entorno real, podría implicar:

* exposición de cuentas válidas,
* acceso a paneles administrativos,
* robo de información,
* compromiso de aplicaciones internas,
* uso posterior de credenciales en otros sistemas.

Aunque Crocodile es una máquina sencilla, representa un caso realista de **encadenamiento de errores de configuración**.

---

# 12. Recomendaciones de mitigación

## 12.1. Deshabilitar acceso anónimo a FTP

Solo debería permitirse acceso autenticado a usuarios explícitamente autorizados.

## 12.2. No almacenar credenciales en texto plano

Los secretos deben:

* evitarse en archivos accesibles,
* almacenarse cifrados o hasheados según corresponda,
* protegerse con permisos estrictos.

## 12.3. Separar credenciales entre servicios

No reutilizar usuarios/contraseñas entre distintos componentes del sistema.

## 12.4. Aplicar principio de mínimo privilegio

Los servicios públicos no deberían exponer archivos sensibles ni directorios innecesarios.

## 12.5. Monitorear accesos no autorizados

Registrar y revisar accesos anónimos o intentos anómalos sobre FTP y paneles web.

---

# 13. Lecciones aprendidas

Este laboratorio deja varias enseñanzas muy valiosas:

* La enumeración inicial define casi todo el camino del ataque.
* FTP sigue siendo relevante cuando está mal configurado.
* Los nombres de archivo ya pueden dar pistas críticas.
* La exposición de credenciales en texto plano es una falla grave.
* Un atacante no siempre necesita explotar software complejo: muchas veces alcanza con aprovechar malas prácticas operativas.

---

# 14. Comandos utilizados

## Escaneo inicial

```bash
nmap -sC -sV -p- -T4 <IP>
```

## Conexión FTP

```bash
ftp <IP>
```

## Listado de archivos remotos

```bash
ls
```

## Descarga de archivos

```bash
get allowed.userlist
get allowed.userlist.passwd
```

## Ver contenido local

```bash
cat allowed.userlist
cat allowed.userlist.passwd
```

---

# 15. Evidencias recomendadas para GitHub

Para que el writeup quede más profesional en tu repositorio, conviene agregar capturas de:

1. Resultado del escaneo con Nmap.
2. Login anónimo exitoso en FTP.
3. Listado de archivos dentro de FTP.
4. Contenido de los archivos descargados.
5. Formulario de login web.
6. Acceso exitoso al panel.
7. Pantalla donde aparece la flag.

---

# 16. Estructura sugerida para el repositorio

```text
HTB/
└── Crocodile/
    ├── README.md
    ├── nmap.txt
    ├── notes.md
    └── screenshots/
        ├── 01-nmap.png
        ├── 02-ftp-login.png
        ├── 03-ftp-files.png
        ├── 04-credentials.png
        ├── 05-web-login.png
        ├── 06-panel.png
        └── 07-flag.png
```

---

# 17. Conclusión final

Crocodile es una máquina sencilla, pero muy útil para entender una idea fundamental del pentesting:

> la combinación de configuraciones inseguras y mala gestión de credenciales puede ser suficiente para comprometer una aplicación completa.

No fue necesario explotar una vulnerabilidad compleja ni desarrollar payloads avanzados. Bastó con:

* enumerar correctamente,
* identificar un servicio mal configurado,
* extraer información sensible,
* y reutilizar esa información en otro vector.

Por eso, este laboratorio es un excelente ejemplo de cómo un atacante puede avanzar rápidamente cuando la organización falla en controles básicos de seguridad.

---

# 18. Resumen ejecutivo

* Se identificaron los servicios **FTP** y **HTTP**.
* El servidor **FTP permitía acceso anónimo**.
* Se descargaron archivos con **usuarios y contraseñas en texto plano**.
* Las credenciales fueron reutilizadas exitosamente en el **login web**.
* Se obtuvo acceso al panel y se recuperó la **flag del laboratorio**.

---

## ✅ Estado final

**Objetivo cumplido: acceso web obtenido mediante reutilización de credenciales expuestas desde un servicio FTP anónimo.**
