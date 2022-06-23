## Pipeline Análisis bioinformático utilizando qiime2

### Paso 1.- 
**1.-** Extraer los archivos obtenidos de la secuenciación (*R1* y *R2*) de sus respectivas carpetas y pasarlos a formato de **qiime2** con el siguiente código:
```
qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' --input-path carpeta_con_archivos --input-format CasavaOneEightSingleLanePerSampleDirFmt --output-path nombre_archivos.qza
```
Estos parametros son para archivos *Paired End*.
También genera el archivo de metadatos con extensión .tsv. Sigue las instrucciones que vienen en **Qiime2** para hacer tu archivo de metadatos.

**1.2-** Crear un archivo con extensión *.qzv* para visualizar la calidad
```
qiime demux summarize --i-data 16s-paired-end.qza --o-visualization 16s-paired-end.qzv 
```

**1.3-**Para visualizar archivos con extensión *.qzv* se usa el siguiente código:

```
qiime tools view nombre_archivo.qzv
```

### Paso 2.- Filtrado y limpieza de secuencias con DADA2
**2.1-** Usamos el plugin dada2 y la función denoise-paired para la limpieza de las secuencias. Como argumentos usamos el archivo de entrada (donde estan las secuencias) y si queremos que corte al principio de las secuencias (--p-trim-left-f y --p-trim-left-r) y al final de las secuencias (--p-trunc-len-f y --p-trunc-len-r). Esta función nos genera 3 archivos de salida: rep-seqs.qza,  table.qza y stats.qza. Adicionalmente podemos agregar el argumento --p-chimera-method y especificar si queremos que quite las quimeras o no ('none').

```
qiime dada2 denoise-paired --i-demultiplexed-seqs its_cucu.qza --p-trim-left-f 0 --p-trim-left-r 0 --p-trunc-len-f 240 --p-trunc-len-r 240 --o-representative-sequences rep-seqs.qza --o-table table.qza --o-denoising-stats stats.qza
```

creamos el archivo de visualización de las *estadisticas*:
```
qiime metadata tabulate --m-input-file stats.qza --o-visualization stats.qzv
```
creamos el archivo de visualización de la *tabla*:
```
qiime feature-table summarize --i-table table.qza --o-visualization table.qzv --m-sample-metadata-file metadata.tsv
```
creamos el archivo de visualización de las *secuencias representativas*:
```
qiime feature-table tabulate-seqs --i-data rep-seqs.qza --o-visualization rep-seqs.qzv
```

### Paso 3 Clusterización
**3.1-** clusterizamos las features para obtener las secuencias representativas, usando el metodo *de novo* con un porcentaje de identidad de 97%.
```
qiime vsearch cluster-features-de-novo --i-sequences rep-seqs.qza --i-table table.qza --p-perc-identity 0.97 --p-threads 20 --o-clustered-table table-cluster-de-novo.qza --o-clustered-sequences seqs-cluster-de-novo.qza
```

### Paso 4 Generar arbol para análisis de diversidad filogenética
El archivo de salida rooted-tree.qza nos va a servir despues para construir el objeto phyloseq para trabajar en R.
```
qiime phylogeny align-to-tree-mafft-fasttree --i-sequences rep-seqs.qza --o-alignment aligned-rep-seqs.qza --o-masked-alignment masked-aligned-rep-seqs.qza --o-tree unrooted-tree.qza --o-rooted-tree rooted-tree.qza
```

### Paso 5 Análisis de diversidad
```
qiime diversity core-metrics-phylogenetic --i-phylogenys rooted-tree.qza --i-table table.qza --p-sampling-depth 1922 --m-metadata-file metadata.tsv --output-dir core-metrics-results
```

### Paso 6 Análisis taxonomico
**5.1-** Realizamos la asignación taxonomica usando alguna de las diferentes bases de datos que podemos obtener. En este ejemplo usamos la base de silva para la región hipervariable V4 del 16s:
```
qiime feature-classifier classify-sklearn --i-classifier silva-138-99-515-806-nb-classifier.qza --i-reads rep-seqs-dada2.qza --o-classification taxonomy-silva.qza
```

**5.2-** Creamos el archivo de visualización de la taxonomía:
```
qiime metadata tabulate --m-input-file taxonomy.qza --o-visualization taxonomy.qzv
```

**5.3-** Creamos un archivo de visualización de la taxonomia en forma de gráfica de barras:
```
qiime taxa barplot --i-table table-dada2.qza --i-taxonomy taxonomy-gg.qza --m-metadata-file 16s_metadata.tsv --o-visualization taxa-bar-plots.qzv
```

## Anexos
Con este comando excluyo del archivo de taxonomia aquellos secuencias que el *Phylum* quedó como sin identificar:
```
qiime taxa filter-seqs --i-sequences rep-seqs.qza --i-taxonomy taxonomy_quimeras.qza --p-include p__ --p-exclude unidentified --o-filtered-sequences sequences-sin-unidentified.qza
```

Con este comando excluyo especificamente aquellas secuencias que el clasificador asigno que pertenecian a mitocondrias y cloroplastos:
```
qiime taxa filter-seqs --i-sequences sequences.qza --i-taxonomy taxonomy.qza --p-include p__ --p-exclude mitochondria,chloroplast --o-filtered-sequences sequences-with-phyla-no-mitochondria-no-chloroplast.qza
```


```
qiime taxa filter-table --i-table table.qza --i-taxonomy taxonomy.qza --p-include p__ --p-exclude mitochondria,chloroplast --o-filtered-table table-with-phyla-no-mitochondria-no-chloroplast.qza
```

Con este comando excluyo aquellos ASV que tienen menos de 10 secuencias de respaldo:
```
qiime feature-table filter-features --i-table table.qza --p-min-frequency 10 --o-filtered-table feature-frecuency-filtered-table.qza
```

Con este comando excluyo aquellos ASV que tienen menos de 10 secuencias de respaldo:
```
qiime feature-table filter-seqs --i-data rep-seqs.qza --i-table table.qza --p-min-frequency 10 --o-filtered-data 
```


