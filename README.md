# 3D Eyes

Generador de autoestereogramas en un solo archivo HTML. Sin dependencias, sin
compilación y sin servidor: se abre con doble clic y funciona.

Un autoestereograma es una imagen plana que esconde una figura tridimensional.
Al mirarla dejando que los ojos enfoquen más allá de la pantalla, el patrón
repetido se fusiona y aparece el relieve.

## Uso

Abre `3d-eyes.html` en un navegador y carga dos imágenes:

- **Imagen de profundidad**: una imagen en escala de grises donde el blanco es
  lo más cercano y el negro lo más lejano. Define el volumen que se va a ver.
- **Textura**: el patrón que se repite en toda la superficie y dentro del cual
  queda escondida la figura.

Cada zona acepta clic, teclado o arrastrar y soltar. Apenas están las dos
imágenes se genera el resultado, y cualquier ajuste posterior lo regenera solo.
El tamaño de salida es el mismo de la imagen de profundidad.

## Controles

### Relieve

| Control | Rango | Por defecto | Qué hace |
| --- | --- | --- | --- |
| Separación ocular | 100 a 700 px | 250 | Distancia entre un punto y su repetición. Es el doble del ancho de franja, y determina también cuánto detalle de textura se conserva. |
| Profundidad de campo | 0,05 a 0,45 | 0,20 | Cuánto se acorta el período en el punto más cercano. Más alto da más relieve. |
| Suavizado del relieve | 0 a 5 px | 0 | Desenfoque de caja sobre el mapa de profundidad. Ablanda los bordes duros. |
| Invertir profundidad | — | apagado | Para imágenes que marcan lo cercano en negro. |
| Aprovechar todo el rango de grises | — | encendido | Lleva el gris más oscuro a negro y el más claro a blanco. Conviene apagarlo si el mapa ya trae un rango intencional. |

La separación ocular se recorta automáticamente al ancho de la imagen cargada,
de modo que siempre quepan al menos tres franjas: con menos, la figura escondida
apenas alcanza a formarse. Cuando eso ocurre, el panel lo avisa.

Una separación alta conviene por dos razones, pero tiene un límite práctico. La
textura conserva más detalle y la deformación acumulada es menor, porque hay
menos franjas. En contra, la fusión divergente exige que el período en pantalla
no supere la distancia entre los ojos, unos 63 mm: en un monitor de 96 ppp eso
son 238 px, así que con separaciones altas conviene mirar la imagen reducida o
en una pantalla de alta densidad.

### Mosaico

La textura ocupa siempre una franja completa a lo ancho, a resolución completa,
así que su tamaño horizontal lo fija la separación ocular. El único ajuste es
**el alto de la repetición vertical**: cada cuántos píxeles vuelve a empezar
hacia abajo. Sigue la proporción original de tu textura mientras no lo toques a
mano, y nunca supera el alto de la imagen.

### Vista

Zoom de 10 % a 800 % con la rueda del mouse o los botones de la barra inferior.
**Ajustar** calza la imagen completa en el área disponible sin ampliar más allá
del tamaño real; **1:1** vuelve al tamaño original. Al ampliar se muestran los
píxeles exactos, y al reducir se interpola, porque el muestreo irregular
dificulta la fusión.

El panel de controles se redimensiona arrastrando su borde derecho, o con las
flechas del teclado cuando el separador tiene el foco.

## Exportación

Botones para PNG y WEBP. El archivo hereda el nombre de la imagen de
profundidad más el sufijo `-estereograma`: cargar `montana.png` produce
`montana-estereograma.png`.

Si el navegador no soporta el formato pedido, la barra de estado lo informa en
vez de entregar un archivo con la extensión equivocada.

## Cómo funciona

La imagen se divide en franjas verticales del ancho de un período. Una de ellas
es la referencia y muestra el patrón sin deformar; cada franja siguiente repite
la anterior desplazada, y la magnitud del desplazamiento en cada punto la dicta
el mapa de profundidad. Al mirar más allá de la pantalla, el cerebro fusiona un
punto con su repetición y deduce de esa distancia a qué profundidad está.

El recorrido va al revés de como suena. Por cada píxel de salida se retrocede
franja por franja hasta caer en la de referencia, y ahí se muestrea la textura:

```
u = x
mientras u esté fuera de la franja de referencia:
    u ∓= franja · (1 − profundidad(u ∓ franja/2) · factor)
color = muestreo bilineal de la textura en u
```

Tres detalles hacen la diferencia entre que se vea bien o mal.

**Todo ocurre en punto flotante.** El desplazamiento no se redondea nunca a
píxeles enteros, así que la profundidad es continua. Redondeándolo, con los
valores por defecto solo cabrían 26 profundidades distintas entre el fondo y el
frente, y el relieve saldría escalonado en bandas visibles.

**La consulta va medio período por delante**, en el sentido de la marcha. La
posición percibida de un par es su punto medio, y ese desfase la hace coincidir
con la columna consultada, de modo que cada punto se ve donde está. Sin él, el
relieve queda corrido y la franja del extremo del mapa no se muestrea nunca.

**La franja de referencia va al centro y su ancho sigue a la profundidad
local**, recalculado en cada fila. Al centro porque la deformación se acumula al
alejarse de ella, y así el recorrido más largo se parte en dos. De ancho
variable porque un píxel justo afuera aterriza a una separación de distancia
mientras uno justo adentro no se mueve: si la franja midiera siempre un período
completo, la diferencia sería una costura vertical, tanto más marcada cuanto más
cerca esté el relieve.

La imagen de profundidad se convierte a luminancia con los coeficientes de la
Rec. 709, de modo que una imagen a color también produce un relieve razonable.

### Lo que este método no hace

No aplica prueba de superficie oculta. Donde la profundidad salta de golpe, el
patrón se corta en vez de ocultarse, y esas costuras diagonales quedan visibles
en los contornos. Es el precio de trabajar en coordenadas continuas, y a cambio
desaparecen los píxeles sueltos que produce el método clásico de imponer pares
de color a distancias enteras.

## Rendimiento

La luminancia se calcula una sola vez al cargar la imagen y se conserva: los
controles de inversión y de rango operan sobre ese resultado en vez de rehacerlo.
Los buffers intermedios se reutilizan entre generaciones y el núcleo escribe
directamente sobre el `ImageData` del lienzo, sin copias intermedias. El
desenfoque usa una suma deslizante, así que su costo no depende del radio, y en
el muestreo de la textura todo lo que solo depende de la fila se calcula una vez
por fila y no una vez por píxel.

El costo crece con la cantidad de franjas, porque cada píxel retrocede tantas
como tenga entre él y la referencia. Sobre una imagen de 1024×768 son unos 90 ms
con la separación por defecto, y menos con separaciones altas.

Mover un slider agenda la regeneración con 140 ms de retardo que se reinician
con cada cambio, de modo que solo se recalcula cuando el usuario se detiene.

## Privacidad

No hay red. Las imágenes se leen con `createImageBitmap`, se procesan en un
`<canvas>` y se descargan mediante un blob local. Nada sale del navegador y no
se guarda nada entre sesiones.

## Requisitos

Un navegador con `createImageBitmap`, `canvas.toBlob` y eventos de puntero.
Chrome, Firefox, Edge y Safari en versiones recientes cumplen. La exportación a
WEBP depende del codificador del navegador.

## Estructura

Un solo archivo. El CSS está en un `<style>` y el JavaScript en un `<script>`,
todo dentro de un closure. Los identificadores están en inglés y el código está
comentado.

Todo el texto que genera el JavaScript vive en el objeto `STRINGS`, con
marcadores con nombre del tipo `{width}`. Para traducir la interfaz hay que
tocar ese objeto y las etiquetas fijas del HTML.

Se incluye además `3d-eyes.min.html`, la misma aplicación minificada, con el
mismo comportamiento y sin comentarios.

## Créditos

La técnica de autoestereograma de imagen única fue publicada por Christopher
Tyler y Maureen Clarke en 1979. La formulación por recorrido inverso que usa
este generador está tomada de
[piellardj/stereogram-webgl](https://github.com/piellardj/stereogram-webgl),
que la implementa sobre GPU y la explica en detalle en su README. Es software
MIT, igual que este.

## Licencia

MIT. Copyright (c) 2026 Sebastián Wilson Traub. El texto completo está en
`LICENSE` y en la cabecera de ambos archivos HTML.
