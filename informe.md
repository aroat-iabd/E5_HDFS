# E5. Integridade dos datos en HDFS
## Primeiros pasos
### Conectarse a Hadoop
Antes hai que copiar o arquivo xera_historico.py á raíz de WSL
```bash
scp xera_historico.py xuedua077@hadoop.cesga.es:
ssh xuedua077@hadoop.cesga.es
```
### Xerar os datos
```bash
python xera_historico.py --mb 64 --saida reservas_historico.csv
```
### Comprobación
```bash
ls
head reservas_historico.csv
```

## 1. A contorna
```bash
hdfs getconf -confKey dfs.blocksize
hdfs getconf -confKey dfs.replication
hdfs getconf -confKey dfs.bytes-per-checksum
hdfs dfs -ls
hdfs dfs -df -h
```

```text
134217728
3
512
*está vacío
Filesystem            Size     Used  Available  Use%
hdfs://nameservice1  2.5 P  283.2 T      2.2 P   11%
```

### Preguntas
#### 1. O tamaño de bloque (en MB), a replicación e cada cantos bytes se calcula unha suma de verificación.
O bloque son 134217728 bytes que se traducen a 128 MB. A replicación é 3 que indica que cada bloque se divide en 3 DataNodes distintos. Calcúlase un CRC cada 512 bytes. 

#### 2. Cantas sumas de verificación ten un bloque completo?


#### 3. Cal é o teu cartafol persoal en HDFS? Canto espazo ten o clúster e canto está ocupado?
O meu cartafol é hdfs://nameservice1

## 2. Subir un ficheiro e ver os seus bloques

### 1. Xera o ficheiro de 64 MB e súbeo dúas veces: unha con bloques de 16 MB e outra co tamaño de bloque por defecto.
```bash
hdfs dfs -mkdir -p e5
hdfs dfs -D dfs.blocksize=16m -put reservas_historico.csv e5/historico_16m.csv
hdfs dfs -put reservas_historico.csv e5/historico_def.csv
hdfs dfs -stat "%n · %b bytes · bloque %o · replicación %r" e5/*
```

### 2. Revisa os bloques e onde están as súas copias
```bash
hdfs fsck /user/<usuario>/e5/historico_16m.csv -files -blocks -locations
hdfs fsck /user/<usuario>/e5/historico_def.csv -files -blocks -locations
```

### 3. Mira canto ocupan
```bash
hdfs dfs -du -h e5
```

### Preguntas
#### 1. Cantos bloques ten cada ficheiro e de que tamaño é cada bloque? Por que o último pode ser máis pequeno?
#### 2. Para un bloque de historico_16m.csv: en que DataNodes están as súas copias? Están todas en DataNodes distintos?
#### 3. Cantos DataNodes ten o clúster? (Búscao no resumo de fsck.)
#### 4. du -h mostra dúas cifras para cada ficheiro. Que significa cada unha?


## 3. Replicación
### 1. Baixa a replicación dun ficheiro a 2 e revisa os bloques
```bash
hdfs dfs -setrep 2 e5/historico_16m.csv
hdfs fsck /user/<usuario>/e5/historico_16m.csv -files -blocks -locations
```

### 2. Agora pide máis copias das que se poden gardar
Un número maior que o de DataNodes do clúster (C2, pregunta 3).

`Non uses a opción ´-w´: quedaría agardando para sempre`
```bash
hdfs dfs -setrep <DataNodes + 1> e5/historico_16m.csv
hdfs fsck /user/<usuario>/e5/historico_16m.csv
```

### 3. Volve deixar a replicación en 3
```bash
hdfs dfs -setrep 3 e5/historico_16m.csv
```

### Preguntas
#### 1. Que cambia en fsck coa replicación 2? Canto ocupa agora o ficheiro segundo du -h?
#### 2. Que mostra fsck ao pedir máis copias que DataNodes? Que significa under-replicated? Por que o estado segue sendo HEALTHY?
#### 3. Que faría o NameNode se un DataNode desaparecese e algúns bloques quedasen con menos copias das debidas?


## 4. Sumas de verificación
### 1. Calcula a suma de verificación dos dous ficheiros
```bash
hdfs dfs -checksum e5/historico_16m.csv e5/historico_def.csv
```

### 2. Repíteo co modo que non depende do tamaño de bloque
```bash
hdfs dfs -Ddfs.checksum.combine.mode=COMPOSITE_CRC -checksum e5/historico_16m.csv e5/historico_def.csv
```

### 3. Fai unha copia dentro de HDFS e compara
```bash
hdfs dfs -cp e5/historico_16m.csv e5/copia.csv
hdfs dfs -checksum e5/historico_16m.csv e5/copia.csv
```

### 4. Cambia un só byte dunha copia local, súbea e compara
```bash
cp reservas_historico.csv modificado.csv
printf 'X' | dd of=modificado.csv bs=1 seek=1000000 conv=notrunc
hdfs dfs -D dfs.blocksize=16m -put modificado.csv e5/modificado.csv
hdfs dfs -checksum e5/historico_16m.csv e5/modificado.csv
```

### 5. Descarga o ficheiro de HDFS e compárao co orixinal
```bash
hdfs dfs -get e5/historico_16m.csv descargado.csv
sha256sum reservas_historico.csv descargado.csv
```

### Preguntas
#### 1. Por que os dous ficheiros do paso 1 teñen distinto resumo se o contido é o mesmo? Fíxate no nome do algoritmo (MD5-of-…MD5-of-512CRC32C): que significa cada número?
#### 2. Que cambia no paso 2? Para que serve ese modo?
#### 3. Por que a copia do paso 3 ten o mesmo resumo?
#### 4. Canto cambiou o resumo no paso 4, tendo en conta que só cambiaches un byte de 64 MB?
#### 5. Por que hai que descargar o ficheiro para comparalo co orixinal, en lugar de comparar hdfs dfs -checksum co sha256sum local?


## 5. Preguntas
### 1. Por que, en Big Data, canto máis volume de datos hai máis importante é comprobar a súa integridade?
### 2. Quen comproba as sumas de verificación ao escribir un bloque, ao lelo e cando ninguén o le?
### 3. Un cliente le un bloque e a súa suma de verificación non coincide. Describe paso a paso que fan o cliente e o NameNode.
### 4. Se todas as copias dun bloque estivesen corruptas, que mostraría fsck? Poderíase recuperar o ficheiro?
### 5. Garage e HDFS usan resumos e copias. Que paper ten cada unha das dúas cousas?


## Para rematar
Borrar os ficheiros do clúster porque é compartido
```bash
hdfs dfs -rm -r -skipTrash e5
```