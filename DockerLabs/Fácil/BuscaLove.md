
## Reconocimiento                         
____
Una vez desplegada comprobamos que tenemos conectividad con ```ping -c1 172.18.0.2```
 
![Pasted image 20250115125510](https://github.com/user-attachments/assets/b74ae2d7-311f-4b55-9a43-21afba33229a)


``-c1`` > solo repite el ping 1 sola vez.

Una vez tenemos conectividad con la maquina Víctima, realizo un reconocimiento con ``Nmap``

```bash
nmap -p- --open -sS -sV -sC -min-rate 5000 -n -Pn 172.18.0.2 -oN Puertos
```

![Pasted image 20250115125915](https://github.com/user-attachments/assets/3bdbf7d7-72a6-4eac-9dcf-dc63f8d2ee19)


Para ver los resultados de una forma mas vistosa realizo un ``cat Puertos -l ruby.rail``

![Pasted image 20250115130025](https://github.com/user-attachments/assets/9ad22776-3b9c-43a9-b286-3b75a7397415)


Como se muestra en el resultado obtenemos los puertos 22 y 80 como puertos abiertos y con sus versiones.

## Puerto 80 (http)
------

Accederemos al servidor web indicando la dirección IP en el buscador del navegador.

![Pasted image 20250117124516](https://github.com/user-attachments/assets/a192f7bc-bac6-4cbf-a566-c96f8a3544d1)


Obtenemos una página por defecto de Apache2.
Revisamos el código fuente de la pagina por si vemos algo fuera de lo normal. Para ello pulsamos ``Crtl + u`` 

![Pasted image 20250117124713](https://github.com/user-attachments/assets/d260d74d-a30b-4573-a0d1-d36383fb5db6)


Revisando todo el código no veo nada relevante.

### Enumeración de directorios (Gobuster)
-------- 
Vista la pagina Web y su código fuente sin obtener ningún resultado llamativo, realizo el siguiente comando con la herramienta GoBuster para encontrar posibles directorios ocultos.

```bash
gobuster dir -u 172.18.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,php,txt -t 200
```

![Pasted image 20250117130341](https://github.com/user-attachments/assets/2d59ab94-db9b-4971-8c02-855fe500c7b0)

<details>
	<summary><strong>Explicación Comando:</strong></summary>
	 gobuster dir > Modo de enumeración de directorios en un servidor web.
	-u 172.18.0.2 > Especificar la dirección IP objetivo donde se va ha enumerar directorios.
	-w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt > especificar el diccionario de palabras para probar nombres de directorios en el servidor objetivo.
	-x html,php,txt > Especificar las extensiones a probar. Ejemplo: index.html, index.php, secret.txt.
	-t 200 > numero de hilos para hacer el escaneo mas rápido. OJO: cuantos mas hilos se pongan mas se puede sobrecargar el servidor objetivo.
</details>

Como se muestra en la imagen, nos encuentra el directorio /wordpress, al cual le voy a pasar nuevamente la herramienta Gobuster para ver posibles directorios ocultos.

```bash
gobuster dir -u 172.18.0.2/wordpress -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,php,txt,sql,py,js -t 200
```

<details>
	<summary><strong>Explicación Comando:</strong></summary>
	 gobuster dir > Modo de enumeración de directorios en un servidor web.
	-u 172.18.0.2/wordpress > Especificar la dirección IP objetivo con el directorio ya encontrado donde se va ha enumerar directorios.
	-w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt > especificar el diccionario de palabras para probar nombres de directorios en el servidor objetivo.
	-x html,php,txt,sql,py,js > Especificar las extensiones a probar. Ejemplo: index.html, index.php, secret.txt.
	-t 200 > numero de hilos para hacer el escaneo mas rápido. OJO: cuantos mas hilos se pongan mas se puede sobrecargar el servidor objetivo.
</details>

![Pasted image 20250117130516](https://github.com/user-attachments/assets/e641cf4a-b429-4255-8d64-823eb7c714d2)


Como resultado, obtenemos un nuevo directorio dentro del directorio wordpress, con nombre index.php.

Examinamos el sitio web con estos directorios que hemos enumerado:

![[Pasted image 20250120121015.png]]

En dicha pagina web no hay nada relevante e incluso los enlaces no llevan a ningún sitio.
Procedemos a examinar el código fuente de la web ``Ctrl + u``: 

![[Pasted image 20250120121124.png]]

En su código fuente solo obtenemos un texto comentado, el cual se resalta en verde.
Como no hemos conseguido nada, simplemente un texto comentado, vamos a usar la herramienta ``WFUZZ`` para ver si existe un LFI (Local file inclusion):

```bash
 wfuzz -c -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt "http://172.18.0.2/wordpress/index.php?FUZZ=../../../../../../../../../../etc/passwd"
```

<details>
	<summary><strong>Explicación Comando:</strong></summary>
	 

wfuzz -> Ejecuta la herramienta Wfuzz, que se usa para pruebas de fuzzing en aplicaciones web.|

-c -> Muestra la salida en color para facilitar la lectura de los resultados.

-w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -> Especifica la **wordlist** que se utilizará para realizar pruebas de inyección en la URL. En este caso, la wordlist contiene nombres de directorios y archivos que podrían existir en el servidor.

"http://172.18.0.2/wordpress/index.php?FUZZ=../../../../../../../../../../etc/passwd" -> Define la **URL objetivo** en la que se ejecutará el fuzzing. La palabra clave `FUZZ` será reemplazada por cada entrada de la wordlist para probar posibles valores.

../../../../../../../../../../etc/passwd -> Intenta realizar una travesía de directorios (Directory Traversal o Path Traversal) para acceder al archivo `/etc/passwd`, que contiene información de los usuarios en sistemas Linux.
</details>

![[Pasted image 20250120121457.png]]

Como vemos en la imagen, nos esta dando muchos resultados, los cuales no son los que queremos obtener, para filtrar este resultado voy a incluir el parámetro ``--hw 115`` que en este caso nos va a excluir todas aquellas respuestas con una cantidad de 115 palabras.

![[Pasted image 20250120122142.png]]

Vemos como nos ha detectado "love" como parámetro a pasar en la URL que hemos usado en la herramienta ``WFUZZ``
Con dicho parámetro, vamos al navegador y ponemos la URL:

![[Pasted image 20250120122348.png]]

Como vemos la pagina es vulnerable a un LFI.
Ahora para verlo mejor abrimos el código fuente ``Ctrl + U``

![[Pasted image 20250120122443.png]]

Como se puede observar, tenemos dos posibles usuario.
pedro y rosa.

Con estos usuarios obtenidos, pasamos a realizar un ataque por fuerza bruta con Hydra al puerto 22 SSH.


## Puerto 22 (SSH):
------ 

Primero usamos ``Hydra`` con el usuario pedro:

```bash
hydra -l pedro -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2
```

<details>
	<summary><strong>Explicación Comando:</strong></summary>
	
hydra -> Ejecuta la herramienta Hydra, que es utilizada para ataques de fuerza bruta contra distintos servicios.
-l pedro -> Especifica el nombre de usuario a probar (`pedro`). Solo se probarán contraseñas para este usuario.
-P /usr/share/wordlists/rockyou.txt -> Usa el archivo `rockyou.txt` como diccionario de contraseñas. Este archivo contiene millones de contraseñas filtradas y es muy usado en pruebas de penetración.|
ssh://172.18.0.2 -> Indica que el ataque será contra el servicio SSH en la IP 172.18.0.2.
</details>

![[Pasted image 20250120123111.png]]

Con el usuario pedro tarda mucho y no, nos esta encontrando nada.

Paso a probar con el usuario rosa empleando el mismo comando de la herramienta ``Hydra``

```bash
hydra -l rosa -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2
```

![[Pasted image 20250120123220.png]]

Para el usuario rosa nos encuentra la contraseña ``lovebug``, la cual vamos a emplear para realizar una conexión SSH.

```bash
ssh rosa@172.18.0.2
```

![[Pasted image 20250120123356.png]]

Estamos dentro como el usuario Rosa

## Escalada de privilegios.
------
Para empezar usaremos el comando ``sudo -l``, de esta manera sabemos que podemos ejecutar como sudo.

![[Pasted image 20250120123553.png]]

Vemos que el usuario rosa puede usar los comando ls (listar archivos) y cat (mostrar contenido de archivos) como root.

Sabiendo esto, procedo a listar todos los archivos y directorios del usuario root, incluyendo los ocultos:

```bash
sudo ls -la /root
```

![[Pasted image 20250120124119.png]]

Obtenemos un archivo secret.txt al cual le pasaremos el comando cat para ver su contenido:

```bash
sudo cat /root/secret.txt
```

![[Pasted image 20250120124224.png]]

Nos muestra un numero hexadecimal, el cual podemos descifrar en la pagina web CyberChef usando el operador ``Magic``:

![[Pasted image 20250120124347.png]]
![[Pasted image 20250120124358.png]]

Nos muestra como resultado: ``noacertarasosi``
La cual puede ser una posible contraseña, en este caso para el usuario pedro.
Pruebo a cambiar de usuario a pedro y introducir dicha contraseña:

![[Pasted image 20250120124538.png]]

Estamos dentro de pedro.
Ahora usamos el comando ``sudo -l``para saber que puede ejecutar pedro como root:

![[Pasted image 20250120124646.png]]

Como vemos, pedro tiene permisos para ejecutar /env con permisos de root.
Visto esto, vamos a buscar en la web CTFOBins una posible escalada de privilegios para /env:

![[Pasted image 20250120124902.png]]

Vamos a la terminal y ejecutamos dicho comando:

```bash
sudo env /bin/sh
```

![[Pasted image 20250120124943.png]]

Ya estaríamos dentro del sistema como el usuario ``ROOT``


