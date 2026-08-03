![[assets/Pasted image 20260803102215.png]]

Metadata

| Field            | Value                    |
| ---------------- | ------------------------ |
| ModifyDate       | 2000:01:01 00:00:00+0000 |
| YCbCrPositioning | Centered                 |
| ExifVersion      | 0210                     |
| DateTimeOriginal | 2000:01:01 00:00:00+0000 |
| CreateDate       | 2000:01:01 00:00:00+0000 |

---

Para acotar el área de búsqueda, el primer paso consiste en determinar la dirección hacia la que apuntaba la cámara. Aunque los metadatos del archivo no aportan información temporal fiable, las pistas visuales del entorno permiten deducir la orientación:

- Momento del dia: La iluminación con tonos dorados y una altura solar baja indican que la fotografía fue tomada durante las últimas horas de la tarde, cerca del anochecer.
- **Estación del año**: La presencia de árboles caducifolios completamente desprovistos de hojas sitúa la escena en el invierno de Canberra (entre los meses de junio y agosto).
- **Análisis Solar**: Durante el invierno austral, el Sol se desplaza a baja altura y se oculta en dirección **Noroeste**.

Para corroborar esto, se utilizó una herramienta de trayectoria solar  ([alpenglowapp](https://alpenglowapp.com)). Al contrastar la iluminación de la escena y la proyección de las sombras con la posición del atardecer, se determinó que la cámara se encuentra orientada hacia el **Suroeste**. 


![[assets/Pasted image 20260803110023.png]]

Al cruzar la orientación obtenida con el trazado urbano de Canberra, se identificó que la única sección de la ciudad que presenta calles alineadas y con vistas hacia edificios en dirección al **Suroeste** es la siguiente:

![[assets/Pasted image 20260803113412.png]]

Con el radio de búsqueda reducido, el siguiente paso consistió en analizar los elementos específicos de la infraestructura urbana presentes en la imagen para lograr una geolocalización exacta:

![[assets/Pasted image 20260803112509.png]]

Un detalle clave en la imagen es el tendido de cables eléctricos que cruza sobre la calle. Dado que este tipo de cableado aéreo es poco común en las zonas modernas de Canberra, su presencia reduce drásticamente el rango de búsqueda a unos pocos puntos específicos. Debido a que estos cables son difíciles de identificar desde la vista aérea satelital, se tomaron como referencias secundarias la señal de tráfico y la posición exacta del árbol debajo de ellos. Tras inspeccionar sistemáticamente los aproximadamente 18 puntos con cableado aéreo en la zona delimitada, se logró localizar el punto exacto:

![[assets/Pasted image 20260803115949.png]]
 [-35.278353,149.1443443]

El sitio localizado coincide con casi la totalidad de los elementos visuales de la imagen original, con una única excepción: la presencia de los semáforos actuales. Mi hipótesis es que la fotografía analizada data aproximadamente del año 2022, un periodo previo a la instalación de esta infraestructura vial por parte del municipio. Dejando de lado esto, todos los demás componentes confirman la geolocalización de forma inequívoca: el diseño del edificio al fondo, la estructura de la casa adyacente, la disposición del cableado, las marcas o huellas en el asfalto y la alineación de la vegetación (incluyendo el segundo árbol, cuyo crecimiento se mantiene idéntico).


