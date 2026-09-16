que es grep: busca en un archivo las líneas que contengan un texto y las muestra

grep -c: no muestra las líneas, solo cuántas coincidieron
grep -w: palabra completa (FAIL sí, FAILED no)
grep -cw: cuántas líneas tienen la palabra completa
grep -v: al revés, las líneas que NO tienen el texto
grep -n: pone el número de línea adelante
grep -i: sin distinguir mayúsculas de minúsculas
grep -E: permite símbolos: ^ empieza con, $ termina con, | o
grep -Ev "^#|^$": quita comentarios y líneas vacías (lo activo de un .conf)

que es cut: corta cada línea en pedazos y devuelve el que pidas
cut -d: -f1: separador ":" y dame el pedazo 1
cut -d' ' -f6-: separador espacio, del pedazo 6 hasta el final

que es sort: ordena las líneas alfabéticamente
sort -n: ordena como números (10 después de 9, no antes)
sort -r: al revés (de mayor a menor)
sort -rn: numérico y de mayor a menor (el "top")
sort -u: sin repetidos

que es uniq -c: junta las líneas repetidas que están PEGADAS y cuenta cuántas eran (por eso va después de sort)

sort | uniq -c | sort -rn: el top de lo más repetido

que es wc -l: cuenta líneas
cat archivo | wc -l: lo mismo pero sin mostrar el nombre del archivo

que es head -3 / tail -3: primeras 3 líneas / últimas 3
que es less: abre un archivo para leerlo con el teclado, no edita (/texto busca, n siguiente, q sale)

que es >: manda la salida a un archivo, pisa lo que había
que es >>: manda la salida al final del archivo, sin pisar
que es 2>: manda los errores a un archivo
que es 2> /dev/null: tira los errores (no se ven)
que es |: la salida de un comando entra al siguiente (tubería)
que es tee: como > pero además muestra en pantalla; es un programa, por eso sudo tee sí escribe en archivos de root
sudo echo "x" > /etc/motd: FALLA, el > lo hace tu shell sin sudo
echo "x" | sudo tee /etc/motd: FUNCIONA

que es /etc/passwd: lista de usuarios, uno por línea, 7 campos con ":" — sin contraseñas (están en /etc/shadow)
que es /etc/motd: mensaje que ve todo el que inicia sesión

que es md5sum: huella del contenido del archivo; misma huella = mismo archivo
que es $?: número que dejó el último comando; 0 bien, otro número falló; se lee justo después con echo $?
