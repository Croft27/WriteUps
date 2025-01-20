## **Reconocimiento**
____
Una vez desplegada la máquina, comprobamos que tenemos conectividad con el siguiente comando:

```bash
ping -c1 172.18.0.2
```

![Pasted image 20250115125510](https://github.com/user-attachments/assets/b74ae2d7-311f-4b55-9a43-21afba33229a)

- **`-c1`** → Ejecuta el **ping** una sola vez.

Una vez confirmada la conectividad con la máquina víctima, realizo un reconocimiento con **Nmap**:

```bash
nmap -p- --open -sS -sV -sC --min-rate 5000 -n -Pn 172.18.0.2 -oN Puertos
```

![Pasted image 20250115125915](https://github.com/user-attachments/assets/3bdbf7d7-72a6-4eac-9dcf-dc63f8d2ee19)

Para visualizar los resultados de una forma más clara, utilizo el siguiente comando:

```bash
cat Puertos -l ruby.rail
```

![Pasted image 20250115130025](https://github.com/user-attachments/assets/9ad22776-3b9c-43a9-b286-3b75a7397415)

Como se muestra en la salida, los **puertos 22 y 80** están abiertos y se muestran sus versiones.

---

## **Puerto 80 (HTTP)**
------

Accedemos al servidor web ingresando la dirección IP en el navegador:

![Pasted image 20250117124516](https://github.com/user-attachments/assets/a192f7bc-bac6-4cbf-a566-c96f8a3544d1)

Obtenemos una página por defecto de **Apache2**.

Revisamos el código fuente de la página en busca de información relevante. Para ello, pulsamos **`Ctrl + U`**:

![Pasted image 20250117124713](https://github.com/user-attachments/assets/d260d74d-a30b-4573-a0d1-d36383fb5db6)

Revisando el código, no encuentro nada de interés.

---

### **Enumeración de directorios con Gobuster**
--------

Después de analizar la página web y su código fuente sin obtener resultados relevantes, utilizo **GoBuster** para encontrar posibles directorios ocultos:

```bash
gobuster dir -u 172.18.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,php,txt -t 200
```

![Pasted image 20250117130341](https://github.com/user-attachments/assets/2d59ab94-db9b-4971-8c02-855fe500c7b0)

<details>
	<summary><strong>Explicación del comando:</strong></summary>

- `gobuster dir` → Modo de enumeración de directorios en un servidor web.  
- `-u 172.18.0.2` → Especifica la dirección IP objetivo donde se enumerarán los directorios.  
- `-w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` → Diccionario de palabras a utilizar.  
- `-x html,php,txt` → Extensiones a probar (ejemplo: `index.html`, `index.php`, `secret.txt`).  
- `-t 200` → Número de hilos para acelerar el escaneo. **Nota:** Cuantos más hilos, mayor carga en el servidor objetivo.  

</details>

Como se muestra en la imagen, encontramos el directorio **/wordpress**, por lo que realizo una nueva búsqueda con **GoBuster** para detectar directorios ocultos dentro de este:

```bash
gobuster dir -u 172.18.0.2/wordpress -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,php,txt,sql,py,js -t 200
```

<details>
	<summary><strong>Explicación del comando:</strong></summary>

- `-u 172.18.0.2/wordpress` → Especifica la ruta del directorio encontrado donde se realizará la enumeración.  
- `-x html,php,txt,sql,py,js` → Extensiones adicionales a probar.  
- `-t 200` → Número de hilos para acelerar el escaneo.  

</details>

![Pasted image 20250120120657](https://github.com/user-attachments/assets/32eec660-f548-4218-9917-1c10c255b42e)

Como resultado, encontramos un **archivo index.php** dentro del directorio `/wordpress`. Examinamos el sitio web con los directorios descubiertos:

![Pasted image 20250120121015](https://github.com/user-attachments/assets/b6a4ad43-d3c8-4d0e-81d8-fcda770cdf1b)

La página no muestra nada relevante y los enlaces no conducen a ningún lugar. Inspeccionamos el código fuente con **`Ctrl + U`**:

![Pasted image 20250120121124](https://github.com/user-attachments/assets/c4671770-59a0-4a48-99f6-dfd48547c3d0)

Solo encontramos un comentario en el código. 
Como no hemos conseguido nada, simplemente un texto comentado, vamos a usar la herramienta WFUZZ para ver si existe un LFI (Local file inclusion).

---

## **Explotación de LFI con WFUZZ**
Realizamos una prueba para detectar **Local File Inclusion (LFI)** con **Wfuzz**:

```bash
wfuzz -c -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt "http://172.18.0.2/wordpress/index.php?FUZZ=../../../../../../../../../../etc/passwd"
```

<details>
	<summary><strong>Explicación del comando:</strong></summary>

- `wfuzz` → Ejecuta la herramienta Wfuzz para fuzzing en aplicaciones web.
- `-c` → Muestra la salida con colores.
- `-w ...` → Especifica la wordlist utilizada para probar valores en `FUZZ`.
- `"http://172.18.0.2/wordpress/index.php?FUZZ=../../../../../../../../../../etc/passwd"` → Intenta un **Path Traversal** para acceder a `/etc/passwd`.

</details>

![Pasted image 20250120121457](https://github.com/user-attachments/assets/405d2ac3-1c34-4489-a14a-c982fdfd8d95)

Filtramos los resultados con `--hw 115` para excluir respuestas de **115 palabras**:

```bash
wfuzz -c -w ... --hw 115 "http://172.18.0.2/wordpress/index.php?FUZZ=../../../../../../../../../../etc/passwd"
```
![Pasted image 20250120122142](https://github.com/user-attachments/assets/f0fe8955-4a46-41e7-851b-baf6864ade0d)

Vemos como nos ha detectado "love" como parámetro a pasar en la URL que hemos usado en la herramienta `WFUZZ` Con dicho parámetro, vamos al navegador y ponemos la URL:

![Pasted image 20250120122348](https://github.com/user-attachments/assets/931d53f8-5733-4344-8a70-d4dfe807fe9c)

Como vemos la pagina es vulnerable a un LFI. Ahora para verlo mejor abrimos el código fuente `Ctrl + U`

![Pasted image 20250120122443](https://github.com/user-attachments/assets/12d59ce3-deb4-4a27-81f9-2dc267e98d98)

Como se puede observar, tenemos dos posibles usuario. pedro y rosa.

Con estos usuarios obtenidos, pasamos a realizar un ataque por fuerza bruta con Hydra al puerto 22 SSH.

---

## **Ataque de fuerza bruta a SSH con Hydra**

Probamos credenciales en el puerto **22 (SSH)** con **Hydra**:

Primero usamos `Hydra` con el usuario pedro:

![Pasted image 20250120123111](https://github.com/user-attachments/assets/07bf9bb4-f8b9-4173-a5a2-daf9bc7b672d)

Con el usuario pedro tarda mucho y no, nos esta encontrando nada.

Paso a probar con el usuario rosa empleando el mismo comando de la herramienta **Hydra**:

```bash
hydra -l rosa -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2
```
![Pasted image 20250120123220](https://github.com/user-attachments/assets/9af24871-f224-4bb4-a0fc-87bed719a33e)

Obtenemos la contraseña **`lovebug`** para el usuario `rosa` y nos conectamos por SSH:

```bash
ssh rosa@172.18.0.2
```
![Pasted image 20250120123356](https://github.com/user-attachments/assets/06dabdde-972f-4840-bf58-871f5860756d)

---

## **Escalada de privilegios**
Ejecutamos:

```bash
sudo -l
```

![Pasted image 20250120123553](https://github.com/user-attachments/assets/e120fca9-9070-4795-9358-6830424c3c47)

El usuario `rosa` puede ejecutar `ls` y `cat` como **root**. Sabiendo esto, procedo a listar todos los archivos y directorios del usuario **root**, incluyendo los ocultos:

```bash
sudo ls -la /root
```
![Pasted image 20250120124119](https://github.com/user-attachments/assets/fb168d44-3492-4f9f-80d0-eebbfbe0e3c8)

Encontramos el archivo `secret.txt`, lo leemos con:

```bash
sudo cat /root/secret.txt
```
![Pasted image 20250120124224](https://github.com/user-attachments/assets/30dd9b50-7e2a-4cf9-b13c-85243c5addd0)

Decodificamos la clave hexadecimal en **CyberChef**, obteniendo la contraseña **"noacertarasosi"** para el usuario `pedro`.

![Pasted image 20250120124347](https://github.com/user-attachments/assets/2fd7b767-017f-4004-b8cb-5fa6581dafd2)
![Pasted image 20250120124358](https://github.com/user-attachments/assets/d0b9b977-3922-4f71-9fef-ab187a2de7c0)

Nos cambiamos a su usuario:

```bash
su pedro
```
![Pasted image 20250120124538](https://github.com/user-attachments/assets/d6bf3d6b-2662-4ab0-9a0b-c5748ee1e943)

Estamos dentro de pedro. Ahora usamos el comando `sudo -l` para saber que puede ejecutar pedro como root:

![Pasted image 20250120124646](https://github.com/user-attachments/assets/4a887f0e-6550-4fba-9e20-15afcb3b2efc)

Como vemos, pedro tiene permisos para ejecutar /env con permisos de root. Visto esto, vamos a buscar en la web CTFOBins una posible escalada de privilegios para /env:

![Pasted image 20250120124902](https://github.com/user-attachments/assets/d48f2737-a4ca-48a6-899f-021d3b452816)

Ejecutamos:

```bash
sudo env /bin/sh
```
![Pasted image 20250120124943](https://github.com/user-attachments/assets/52ce355a-5ddc-4afc-a770-3c45b4c90d98)

✅ ¡Hemos escalado a **root**!

---
