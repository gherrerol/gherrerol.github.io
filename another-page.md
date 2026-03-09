---
layout: default
---

# Práctica 1: Follow Line

En esta primera práctica de robótica, se busca implementar una solución para el problema de Unibotics: Follow Line. La práctica consiste en conseguir que el robot (en este caso un coche de fórmula 1), siga la línea roja pintada sobre la carretera de varios circuitos, recibiendo la imagen únicamente a través de una cámara frontal. 

El reto no es solo seguir la línea, sino hacerlo de manera estable y rápida ante condiciones variables (rectas, curvas de distinta apertuda y transiciones entre ambas). Para ello adoptamos el enfoque del control reactivo cerrado, donde la señal de control (el ángulo de giro del vehículo) se calcula en cada frame a partir del error visual entre la posición de la línea y el centro de la imagen.

A continuación, comentaré en secciones cómo se ha llevado a cabo el desarrollo de la práctica, la implementación final y los problemas encontrados, empezando por el preprocesado de la imagen.

## 1. Preprocesamiento

Lo primero que necesitamos para que el robot vea la línea roja a seguir es definir un espacio de color para aislarla del resto de cosas que vemos en la imagen. Para ello, usamos un rango HSV para definir los valores del color rojo. Usamos el espacio HSV ya que es más robusto frente a cambios de iluminación y usamos la suma de dos rangos de color rojo ya que HSV no tiene un rango continuo, aparece tanto en la zona baja (H: 0-10) como en la alta (H: 160-180). De esta manera, nos aseguramos de que nuestra tonalidad de rojo está siendo capturada.

| Rango                | Resultado                                              |
|:---------------------|:-------------------------------------------------------|
| Bajo 0-10            | Detección en zonas parcialmente oscuras                | 
| Alto 160-180         | Detección en zonas parcialmente iluminadas             | 
| Combinación de ambos | Detección completa y robusta durante todo el recorrido | 

El resultado es una imagen binaria donde los pixeles blancos corresponden a la línea roja. Sobre esta máscara se calcula el centroide mediante momentos de imagen de OpenCV, obteniendo la coordenada X del punto medio de la línea en cada zona analizada.

En esta sección no hubo muchos problemas, el cálculo de la máscara se hizo realizando una búsqueda del color rojo sobre el espacio HSV y funcionó a la primera. El debug de los centroides también funcionó correctamente.

## 2. Detección multi-zona y lookahead




[back](./)
