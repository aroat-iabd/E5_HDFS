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
O bloque son 134217728 bytes que se traducen a 128 MB. A replicación é 3 que indica que cada bloque se replica enteiro en 3 DataNodes distintos. Calcúlase un CRC cada 512 bytes. 

#### 2. Cantas sumas de verificación ten un bloque completo?
$\frac{134\,217\,728\text{ bytes}}{512\text{ bytes/suma}} = \mathbf{262\,144}$  sumas de verificación por bloque completo.

#### 3. Cal é o teu cartafol persoal en HDFS? Canto espazo ten o clúster e canto está ocupado?
- ``hdfs://nameservice1`` é o identificador do sistema de ficheiros raíz (o nameservice), non o meu cartafol persoal. O meu cartafol persoal en HDFS é ``/user/xuedua077`` (é o que lista por defecto hdfs dfs -ls).
- O clúster ten 2.5 PB de capacidade total, dos cales están ocupados 283.2 TB (11 %) e quedan 2.2 PB dispoñibles.

## 2. Subir un ficheiro e ver os seus bloques

### 1. Xera o ficheiro de 64 MB e súbeo dúas veces: unha con bloques de 16 MB e outra co tamaño de bloque por defecto.
```bash
hdfs dfs -mkdir -p e5
hdfs dfs -D dfs.blocksize=16m -put reservas_historico.csv e5/historico_16m.csv
hdfs dfs -put reservas_historico.csv e5/historico_def.csv
hdfs dfs -stat "%n · %b bytes · bloque %o · replicación %r" e5/*
```
```text
historico_16m.csv ?? 67108925 bytes ?? bloque 16777216 ?? replicaci??n 3
historico_def.csv ?? 67108925 bytes ?? bloque 134217728 ?? replicaci??n 3
```
### 2. Revisa os bloques e onde están as súas copias
```bash
hdfs fsck /user/xuedua077/e5/historico_16m.csv -files -blocks -locations
hdfs fsck /user/xuedua077/e5/historico_def.csv -files -blocks -locations
```
```text
0. BP-562833966-10.121.14.18-1548250294268:blk_1088370453_14664471 len=16777216 Live_repl=3
[DatanodeInfoWithStorage[10.121.14.10:9866,DS-17628fef-a754-4509-8161-c4e43285a545,DISK],
DatanodeInfoWithStorage[10.121.14.12:9866,DS-d34abb81-4194-4069-886e-1688b49db705,DISK], 
DatanodeInfoWithStorage[10.121.14.11:9866,DS-d3328556-c172-43d8-a144-557155bdce41,DISK]]

1. BP-562833966-10.121.14.18-1548250294268:blk_1088370454_14664472 len=16777216 Live_repl=3
[DatanodeInfoWithStorage[10.121.14.4:9866,DS-72130375-deb1-4e10-af9c-03bbc083b970,DISK],
DatanodeInfoWithStorage[10.121.14.8:9866,DS-8f6fc2af-08eb-4d44-b5d6-04eebe1528b8,DISK],
DatanodeInfoWithStorage[10.121.14.14:9866,DS-84e6cc9f-90b5-4f61-a620-626ad44df874,DISK]]

2. BP-562833966-10.121.14.18-1548250294268:blk_1088370455_14664473 len=16777216 Live_repl=3
[DatanodeInfoWithStorage[10.121.14.6:9866,DS-82057f43-5d7c-4968-b900-6f43ac8cccba,DISK],
DatanodeInfoWithStorage[10.121.14.1:9866,DS-9360aa84-70f4-4b46-8fef-b1bb368e0bf2,DISK],
DatanodeInfoWithStorage[10.121.14.3:9866,DS-c7a4d2e7-3533-4a5b-9d0c-36dd18b80130,DISK]]

3. BP-562833966-10.121.14.18-1548250294268:blk_1088370456_14664474 len=16777216 Live_repl=3
[DatanodeInfoWithStorage[10.121.14.10:9866,DS-d34a50d6-4a47-4225-a2ff-87584efc8773,DISK],
DatanodeInfoWithStorage[10.121.14.3:9866,DS-cef69aab-1919-4780-9a18-66ae0fca7730,DISK],
DatanodeInfoWithStorage[10.121.14.14:9866,DS-c0b7e71e-1f7d-44aa-abd3-37857d54e428,DISK]]

4. BP-562833966-10.121.14.18-1548250294268:blk_1088370457_14664475 len=61 Live_repl=3
[DatanodeInfoWithStorage[10.121.14.6:9866,DS-7dda0e7d-d778-4e99-b946-645f13bb0efc,DISK],
DatanodeInfoWithStorage[10.121.14.11:9866,DS-7385723e-c95f-4fce-b5ad-83aa07089422,DISK],
DatanodeInfoWithStorage[10.121.14.12:9866,DS-b82cf841-96c0-43b8-8949-dee66cb21d17,DISK]]

Status: HEALTHY
 Number of data-nodes:  14
 Number of racks:               1
 Total dirs:                    0
 Total symlinks:                0
```
```text
0. BP-562833966-10.121.14.18-1548250294268:blk_1088370458_14664476 len=67108925 Live_repl=3 
[DatanodeInfoWithStorage[10.121.14.11:9866,DS-81c485e3-7fd9-42bd-936f-6a2e45ed4965,DISK],
DatanodeInfoWithStorage[10.121.14.3:9866,DS-5df6dcc3-ad81-4dbf-8cff-9d0e572032cf,DISK],
DatanodeInfoWithStorage[10.121.14.14:9866,DS-072d1e4c-44a8-4b8c-aff6-936d70246bff,DISK]
```

### 3. Mira canto ocupan
```bash
hdfs dfs -du -h e5
```
```text
64.0 M  192.0 M  e5/historico_16m.csv
64.0 M  192.0 M  e5/historico_def.csv
```

### Preguntas
#### 1. Cantos bloques ten cada ficheiro e de que tamaño é cada bloque? Por que o último pode ser máis pequeno?
O primeiro ten 4 bloques de 16MB e o último é de 61 bytes que son os datos restantes, o segundo ten un solo bloque de 64MB.
O último bloque é máis pequeno porque HDFS solo almacena os bytes restantes ata o final do ficheiro, sen encher con ceros ata o tamaño máximo de bloque.
#### 2. Para un bloque de historico_16m.csv: en que DataNodes están as súas copias? Están todas en DataNodes distintos?
O bloque 0 (``blk_1088370453_14664471``) está nos DataNodes ``10.121.14.10``, ``10.121.14.12`` e ``10.121.14.11``. Si, están en DataNodes distintos (as 3 réplicas sempre van a nodos diferentes para que a caída dun solo DataNode non faga perder dúas copias).
#### 3. Cantos DataNodes ten o clúster? (Búscao no resumo de fsck.)
Ten 14 DataNodes.
#### 4. du -h mostra dúas cifras para cada ficheiro. Que significa cada unha?
- A primeira cifra (``64.0 M``) é o tamaño lóxico do ficheiro (os datos reais sen réplicas).
- A segunda cifra (``192.0 M``) é o espazo físico total ocupado no clúster contando todas as réplicas ($64.0\text{ MB} \times 3 = 192.0\text{ MB}$).

## 3. Replicación
### 1. Baixa a replicación dun ficheiro a 2 e revisa os bloques
```bash
hdfs dfs -setrep 2 e5/historico_16m.csv
hdfs fsck /user/xuedua077/e5/historico_16m.csv -files -blocks -locations
hdfs dfs -du -h e5
```
```text
Replication 2 set: e5/historico_16m.csv
```
```text
0. BP-562833966-10.121.14.18-1548250294268:blk_1088370453_14664471 len=16777216 Live_repl=2  [DatanodeInfoWithStorage[10.121.14.10:9866,DS-17628fef-a754-4509-8161-c4e43285a545,DISK], DatanodeInfoWithStorage[10.121.14.11:9866,DS-d3328556-c172-43d8-a144-557155bdce41,DISK]]

1. BP-562833966-10.121.14.18-1548250294268:blk_1088370454_14664472 len=16777216 Live_repl=2  [DatanodeInfoWithStorage[10.121.14.4:9866,DS-72130375-deb1-4e10-af9c-03bbc083b970,DISK], DatanodeInfoWithStorage[10.121.14.8:9866,DS-8f6fc2af-08eb-4d44-b5d6-04eebe1528b8,DISK]]

2. BP-562833966-10.121.14.18-1548250294268:blk_1088370455_14664473 len=16777216 Live_repl=2  [DatanodeInfoWithStorage[10.121.14.3:9866,DS-c7a4d2e7-3533-4a5b-9d0c-36dd18b80130,DISK], DatanodeInfoWithStorage[10.121.14.1:9866,DS-9360aa84-70f4-4b46-8fef-b1bb368e0bf2,DISK]]

3. BP-562833966-10.121.14.18-1548250294268:blk_1088370456_14664474 len=16777216 Live_repl=2  [DatanodeInfoWithStorage[10.121.14.10:9866,DS-d34a50d6-4a47-4225-a2ff-87584efc8773,DISK], DatanodeInfoWithStorage[10.121.14.14:9866,DS-c0b7e71e-1f7d-44aa-abd3-37857d54e428,DISK]]

4. BP-562833966-10.121.14.18-1548250294268:blk_1088370457_14664475 len=61 Live_repl=2  [DatanodeInfoWithStorage[10.121.14.12:9866,DS-b82cf841-96c0-43b8-8949-dee66cb21d17,DISK], DatanodeInfoWithStorage[10.121.14.11:9866,DS-7385723e-c95f-4fce-b5ad-83aa07089422,DISK]]

Status: HEALTHY
 Number of data-nodes:  14
 Number of racks:               1
 Total dirs:                    0
 Total symlinks:                0

Replicated Blocks:
 Total size:    67108925 B
 Total files:   1
 Total blocks (validated):      5 (avg. block size 13421785 B)
 Minimally replicated blocks:   5 (100.0 %)
 Over-replicated blocks:        0 (0.0 %)
 Under-replicated blocks:       0 (0.0 %)
 Mis-replicated blocks:         0 (0.0 %)
 Default replication factor:    3
 Average block replication:     2.0
 Missing blocks:                0
 Corrupt blocks:                0
 Missing replicas:              0 (0.0 %)
```
```text
64.0 M  128.0 M  e5/historico_16m.csv
64.0 M  192.0 M  e5/historico_def.csv
```

### 2. Agora pide máis copias das que se poden gardar
Un número maior que o de DataNodes do clúster (C2, pregunta 3).

`Non uses a opción ´-w´: quedaría agardando para sempre`
```bash
hdfs dfs -setrep 15 e5/historico_16m.csv
hdfs fsck /user/xuedua077/e5/historico_16m.csv
```
```text
Replication 15 set: e5/historico_16m.csv
```
```text

/user/xuedua077/e5/historico_16m.csv:  Under replicated BP-562833966-10.121.14.18-1548250294268:blk_1088370453_14664471. Target Replicas is 15 but found 14 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

/user/xuedua077/e5/historico_16m.csv:  Under replicated BP-562833966-10.121.14.18-1548250294268:blk_1088370454_14664472. Target Replicas is 15 but found 14 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

/user/xuedua077/e5/historico_16m.csv:  Under replicated BP-562833966-10.121.14.18-1548250294268:blk_1088370455_14664473. Target Replicas is 15 but found 14 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

/user/xuedua077/e5/historico_16m.csv:  Under replicated BP-562833966-10.121.14.18-1548250294268:blk_1088370456_14664474. Target Replicas is 15 but found 14 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

/user/xuedua077/e5/historico_16m.csv:  Under replicated BP-562833966-10.121.14.18-1548250294268:blk_1088370457_14664475. Target Replicas is 15 but found 14 live replica(s), 0 decommissioned replica(s), 0 decommissioning replica(s).

Status: HEALTHY
 Number of data-nodes:  14
 Number of racks:               1
 Total dirs:                    0
 Total symlinks:                0

Replicated Blocks:
 Total size:    67108925 B
 Total files:   1
 Total blocks (validated):      5 (avg. block size 13421785 B)
 Minimally replicated blocks:   5 (100.0 %)
 Over-replicated blocks:        0 (0.0 %)
 Under-replicated blocks:       5 (100.0 %)
 Mis-replicated blocks:         0 (0.0 %)
 Default replication factor:    3
 Average block replication:     14.0
 Missing blocks:                0
 Corrupt blocks:                0
 Missing replicas:              5 (6.6666665 %)
```

### 3. Volve deixar a replicación en 3
```bash
hdfs dfs -setrep 3 e5/historico_16m.csv
```
```text
Replication 3 set: e5/historico_16m.csv
```

### Preguntas
#### 1. Que cambia en fsck coa replicación 2? Canto ocupa agora o ficheiro segundo du -h?
- En ``fsck``, cada bloque pasa a ter ``Live_repl=2`` (en lugar de 3), aparecen só 2 DataNodes por bloque e no resumo figura ``Average block replication: 2.0``.
- Segundo ``du -h``, o tamaño lóxico mantense en **64.0 M**, pero o espazo físico ocupado no clúster baixa de **192.0 M** a **128.0 M** ($64\text{ MB} \times 2$).
#### 2. Que mostra fsck ao pedir máis copias que DataNodes? Que significa under-replicated? Por que o estado segue sendo ``HEALTHY``?
- Que mostra ``fsck``: Indica para os 5 bloques ``Under replicated ... Target Replicas is 15 but found 14 live replica(s)``, e no resumo marca ``Under-replicated blocks: 5 (100.0 %), Average block replication: 14.0`` e ``Missing replicas: 5 (6.6666665 %)``.
- Que significa *under-replicated*: Que os bloques teñen menos réplicas activas (14) das que se solicitaron como obxectivo (15), porque HDFS nunca garda dúas copias do mesmo bloque nun mesmo DataNode e só hai 14 nodos dispoñibles.
- Por que segue sendo **``HEALTHY``**: Porque ``Missing blocks: 0`` e ``Corrupt blocks: 0``; isto é, todos os bloques existen, teñen polo menos unha copia sa (de feito teñen 14) e os datos pódense ler sen perda.
#### 3. Que faría o NameNode se un DataNode desaparecese e algúns bloques quedasen con menos copias das debidas?
Ao deixar de recibir *heartbeats* dese DataNode, o NameNode márcao como morto, detecta que os bloques que gardaba quedaron *under-replicated* e ordena automaticamente aos DataNodes que conservan réplicas sas que as copien cara a outros DataNodes activos do clúster ata recuperar o factor de replicación configurado.

## 4. Sumas de verificación
### 1. Calcula a suma de verificación dos dous ficheiros
```bash
hdfs dfs -checksum e5/historico_16m.csv e5/historico_def.csv
```
```text
e5/historico_16m.csv    MD5-of-32768MD5-of-512CRC32C    0000020000000000000080006ae2f7f2b5c099a73b4a2eed8ab700ed
e5/historico_def.csv    MD5-of-0MD5-of-512CRC32C        000002000000000000000000b349a5b4908740c0dfa3a87336a9846b
```

### 2. Repíteo co modo que non depende do tamaño de bloque
```bash
hdfs dfs -Ddfs.checksum.combine.mode=COMPOSITE_CRC -checksum e5/historico_16m.csv e5/historico_def.csv
```
```text
e5/historico_16m.csv    COMPOSITE-CRC32C        5f0623f9
e5/historico_def.csv    COMPOSITE-CRC32C        5f0623f9
```

### 3. Fai unha copia dentro de HDFS e compara
```bash
hdfs dfs -cp e5/historico_16m.csv e5/copia.csv
hdfs dfs -checksum e5/historico_16m.csv e5/copia.csv
```
```text
e5/historico_16m.csv    MD5-of-32768MD5-of-512CRC32C    0000020000000000000080006ae2f7f2b5c099a73b4a2eed8ab700ed
e5/copia.csv    MD5-of-0MD5-of-512CRC32C        000002000000000000000000b349a5b4908740c0dfa3a87336a9846b
```

### 4. Cambia un só byte dunha copia local, súbea e compara
```bash
cp reservas_historico.csv modificado.csv
printf 'X' | dd of=modificado.csv bs=1 seek=1000000 conv=notrunc
hdfs dfs -D dfs.blocksize=16m -put modificado.csv e5/modificado.csv
hdfs dfs -checksum e5/historico_16m.csv e5/modificado.csv
```
```text
1+0 records in
1+0 records out
1 byte (1 B) copied, 0.000411253 s, 2.4 kB/s
```
```text
e5/historico_16m.csv    MD5-of-32768MD5-of-512CRC32C    0000020000000000000080006ae2f7f2b5c099a73b4a2eed8ab700ed
e5/modificado.csv       MD5-of-32768MD5-of-512CRC32C    00000200000000000000800002b51d0ccda7017aa5616b99f7dde5f4
```

### 5. Descarga o ficheiro de HDFS e compárao co orixinal
```bash
hdfs dfs -get e5/historico_16m.csv descargado.csv
sha256sum reservas_historico.csv descargado.csv
```
```text
43647ae2b81ea1ba02edc3819f81327872c769db91e9ccf30d0677e2cff34891  reservas_historico.csv
43647ae2b81ea1ba02edc3819f81327872c769db91e9ccf30d0677e2cff34891  descargado.csv
```

### Preguntas
#### 1. Por que os dous ficheiros do paso 1 teñen distinto resumo se o contido é o mesmo? Fíxate no nome do algoritmo (MD5-of-…MD5-of-512CRC32C): que significa cada número?
Teñen distinto resumo porque o algoritmo por defecto depende de como se agrupan os bloques.
- **``512``**: calcúlase un ``CRC32C`` cada 512 bytes de datos.
- **``32768``** (en ``historico_16m.csv``): é o número de CRCs que caben nun bloque de 16 MB ($\frac{16\,777\,216}{512} = 32\,768$), aos que se lles aplica un ``MD5`` por bloque. En ``historico_def.csv`` aparece **``0``** porque o ficheiro cabe enteiro nun único bloque menor ao límite de 128 MB.
- **Primeiro ``MD5``**: é o ``MD5`` final calculado sobre a concatenación dos ``MD5`` de cada bloque. Ao variar o tamaño de bloque, cambia a agrupación e o resumo final é distinto.

#### 2. Que cambia no paso 2? Para que serve ese modo?
Co modo ``COMPOSITE_CRC``, ambos os ficheiros devolven exactamente o mesmo resumo (``COMPOSITE-CRC32C  5f0623f9``). Serve para obter unha suma de verificación global baseada unicamente no contido, independente do tamaño de bloque, permitindo comprobar se dous ficheiros son idénticos aínda que estean divididos en bloques distintos.

#### 3. Por que a copia do paso 3 ten o mesmo resumo?
Na miña execución, ``e5/copia.csv`` non deu o mesmo resumo ca ``historico_16m.csv``, senón que deu exactamente o mesmo ca ``historico_def.csv`` (``MD5-of-0MD5-of-512CRC32C ...b349a5b4908740c0dfa3a87336a9846b``). Isto ocorre porque ``hdfs dfs -cp`` sen parámetros adicionais creou a copia co tamaño de bloque por defecto do clúster (128 MB) en lugar dos 16 MB do orixinal. O contido é o mesmo e, se conservase o bloque de 16 MB (ou usando ``COMPOSITE_CRC``), o resumo sería idéntico.

#### 4. Canto cambiou o resumo no paso 4, tendo en conta que só cambiaches un byte de 64 MB?
Os primeiros 24 díxitos hexadecimais (``000002000000000000008000``) mantéñense porque representan os metadatos de estrutura (512 bytes/CRC e 32768 CRCs/bloque), pero o hash MD5 final cambiou por completo (``6ae2f7f2b5c099a73b4a2eed8ab700ed`` fronte a ``02b51d0ccda7017aa5616b99f7dde5f4``). Isto débese ao efecto avalancha das funcións hash: cambiar 1 byte altera o CRC32C dese fragmento de 512 bytes, iso cambia o MD5 dese bloque e transforma todo o MD5 final.

#### 5. Por que hai que descargar o ficheiro para comparalo co orixinal, en lugar de comparar hdfs dfs -checksum co sha256sum local?
Porque usan algoritmos incompatibles (``MD5-of-...-CRC32C`` baseado na estrutura de bloques de HDFS fronte a ``SHA-256`` lineal sobre todo o ficheiro) e o sistema de ficheiros local non garda os metadatos de bloques de HDFS. Ao descargalo, pódese executar exactamente a mesma función (``sha256sum``) sobre os dous ficheiros locais e verificar que coinciden (``43647ae2b81ea1ba02edc3819f81327872c769db91e9ccf30d0677e2cff34891``).

## 5. Preguntas
### 1. Por que, en Big Data, canto máis volume de datos hai máis importante é comprobar a súa integridade?
Cando se almacenan terabytes ou petabytes repartidos en ducias ou centos de discos e nodos, os fallos de hardware, erros de rede e a corrupción silenciosa en disco (*bit rot*) deixan de ser casos illados e ocorren de xeito continuo. Sen verificación automática, os datos corromperíanse sen aviso e alterarían os resultados das análises.

### 2. Quen comproba as sumas de verificación ao escribir un bloque, ao lelo e cando ninguén o le?
- **Ao escribir:** os propios **DataNodes** do *pipeline* verifican as sumas antes de gardar o bloque en disco.
- **Ao ler:** o **cliente HDFS** calcula a suma de verificación dos datos que recibe e compáraa coa almacenada polo DataNode.
- **Cando ninguén o le:** o **DataBlockScanner** (un proceso en segundo plano en cada DataNode) revisa periodicamente os bloques no disco para detectar corrupción silenciosa antes de que ningún cliente os solicite.

### 3. Un cliente le un bloque e a súa suma de verificación non coincide. Describe paso a paso que fan o cliente e o NameNode.
1. O cliente detecta que a suma calculada non coincide coa gardada e lanza unha excepción (``ChecksumException``).
2. O cliente avisa ao **NameNode** indicando que esa réplica dese bloque nese DataNode está corrupta.
3. O cliente solicita outro DataNode que teña outra réplica do bloque e continúa a lectura desde el.
4. O NameNode marca a réplica defectuosa como corrupta, ordena copiar unha nova réplica desde un DataNode san cara a outro nodo activo para manter o factor de replicación e manda borrar a réplica corrupta.

### 4. Se todas as copias dun bloque estivesen corruptas, que mostraría fsck? Poderíase recuperar o ficheiro?
``fsck`` mostraría o estado **``CORRUPT``** (indicando bloques corruptos ou perdidos). Non se podería recuperar o ficheiro dentro de HDFS porque non quedaría ningunha copia íntegra dese bloque; só se podería restaurar mediante unha copia de seguridade externa.

### 5. Garage e HDFS usan resumos e copias. Que paper ten cada unha das dúas cousas?
- Os **resumos (checksums/hashes)** garanten a **integridade**: permiten detectar se un bloque ou obxecto sufriu alteracións ou corrupción.
- As **copias (replicación)** garanten a **dispoñibilidade e a recuperación**: permiten seguir accedendo aos datos se cae un nodo e reparar calquera réplica corrupta substituíndoa por outra sa.

## Para rematar
Borrar os ficheiros do clúster porque é compartido
```bash
hdfs dfs -rm -r -skipTrash e5
```
```text
Deleted e5
```