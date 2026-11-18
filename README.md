# AnalisisDeDatos
# Prediccion de congestion en un enlace LAN

Laboratorio de analisis de datos en redes. Captura de trafico UDP con iperf3

## Contenido del repositorio

```
README.md
informe/     informe en PDF
capturas/    pantallazos del montaje y de las pruebas
codigo/      scripts de captura, construccion de ventanas, modelo y graficas
datos/       por corrida: traza pcap, registro de cola, CSV de paquetes, JSON de iperf3 y calendario
resultados/  CSV de ventanas, metricas y graficas temporales
```

## 1. Topologia y montaje

Se usaron dos equipos en la misma red del router: un emisor Linux (A) que aplica la cola y hace la captura, y un receptor (B) que ejecuta el servidor de iperf3.

| Equipo | Rol | Sistema | Direccion IP | Software |
|---|---|---|---|---|
| A | Emisor, cola TBF, captura y muestreo | Ubuntu Server 26.04 LTS en una maquina virtual de VirtualBox, sobre un PC con Windows | 192.168.20.61  | iperf3 (cliente), tc, tcpdump, tshark, Python con pandas y scikit-learn |
| B | Receptor | MacBook con macOS | 192.168.20.189 | iperf3 (servidor), compilado desde el codigo fuente |

La VM tiene dos adaptadores. El adaptador NAT solo se uso para instalar paquetes y conectarse por SSH, un adaptador en modo puente conectado a la red del router, y en esa interfaz se aplica la cola.

El MacBook no tiene puerto Ethernet, asi que B se conecto por WiFi al mismo router. Esto difiere del enlace Ethernet directo del enunciado y se discute al final.


Interfaces de A antes de configurar enp0s8:

![Interfaces de A](<img width="1015" height="91" alt="01_interfaces_iniciales" src="https://github.com/user-attachments/assets/6f155584-6df2-47e5-bdda-deb6e4738298" />
)

Configuracion de enp0s8 con netplan usando la IP fija del enunciado (192.168.50.10/24). Despues enp0s8 quedo en modo puente con la red del router, con la IP 192.168.20.61 que se uso en las corridas:

![Netplan en A](capturas/02_netplan_ip_fija.png)

Verificacion de conectividad con ping y prueba corta de iperf3 a 8 Mbit/s desde A hacia B:

![Ping e iperf3 en A](capturas/03_ping_iperf3_cliente_A.png)

Salida del servidor iperf3 en B para esa misma prueba:

![Servidor iperf3 en B](capturas/04_iperf3_servidor_B.png)

## 2. Protocolo y perfiles de carga

Se hicieron seis corridas independientes de entre 110 y 205 s, con cola, captura y muestreo continuos, en las que A envia trafico UDP a B por el puerto 5201.

**Cola.** En enp0s8 se aplico una cola TBF de 10 Mbit/s antes de cada corrida, con el contador de descartes en cero. El enunciado propone burst de 16 kb y limite de 32 kb; las corridas finales usaron burst de 32 kb y limite de 128 kb (ver desviaciones).

```
tc qdisc add dev enp0s8 root handle 10: tbf rate 10mbit burst 32kb limit 128kb
```

**Trafico.** Cada fase es un cliente de iperf3 en UDP con datagramas de 1200 bytes, intervalos de 1 s y salida en JSON. Se uso un buffer de envio de 4 MB (`-w 4M`) con `net.core.wmem_max` en 8388608; sin el, el emisor se frenaba solo y nunca llenaba la cola.

```
iperf3 -c 192.168.20.189 -u -b <tasa>M -l 1200 -t <segundos> -i 1 -w 4M --get-server-output -J
```

**Captura.** tcpdump en enp0s8 con `-s 96 -B 4096`, filtrando UDP del emisor al receptor en el puerto 5201. Se exporto con tshark la hora y la longitud de trama de cada paquete. En el ensayo de verificacion tcpdump capturo 13629 paquetes y el kernel no descarto ninguno. Las perdidas de captura de cada corrida final quedan registradas en su archivo de resumen de tcpdump.

**Muestreo de la cola.** Cada segundo, `tc -j -s qdisc show` guarda el contador acumulado de descartes de la cola `tbf 10:`, con su hora Unix.

**Calendario.** Cada corrida guarda en un CSV la hora de inicio y de fin de cada fase. Se usa para interpretar los resultados y no entra como variable del modelo. Si iperf3 fallaba al empezar una fase (servidor ocupado), el script reintentaba.

| Corrida | Perfil (Mbit/s : segundos) | Que contiene | Ventanas de 1 s | Uso |
|---|---|---|---|---|
| c01 | 4:20, 8:20, 14:20, 5:20, 8:20, 14:20, 5:20 | Dos episodios; saltos de 8 a 14 | 136 | Entrenamiento |
| c02 | 4:20, 6:15, 8:15, 10:15, 12:15, 14:20, 5:20, 4:15, 8:15, 11:15, 14:20, 5:20 | Dos ascensos graduales de 4 a 14 | 180 | Entrenamiento |
| c03 | 5:30, 14:20, 4:25, 14:15, 5:25, 14:25, 4:20 | Tres episodios con saltos bruscos | 152 | Entrenamiento |
| c04 | 4:20, 9:20, 11:20, 13:20, 5:20, 9:20, 11:20, 13:20, 5:20 | Dos ascensos en escalon de 9 a 13 | 166 | Validacion |
| c05 | 5:20, 14:10, 5:20, 14:10, 5:20, 14:10, 5:20 | Tres episodios cortos con saltos de 5 a 14 | 100 | Prueba |
| c06 | 3:20, 7:10, 9:10, 11:10, 14:25, 6:15, 10:10, 12:10, 15:20, 4:20 | Dos rampas y un salto a 14 | 135 | Prueba |

En total hay 869 ventanas y unos 625 mil paquetes capturados. Las tasas son carga UDP ofrecida por iperf3; la tasa observada en la interfaz puede diferir por la sobrecarga de los protocolos.

## 3. Ventanas, variables y modelo

Cada fila del conjunto de datos es una ventana de unos 1,05 s entre dos muestras consecutivas de la cola, con la tasa capturada como predictor y los descartes de la ventana siguiente como objetivo.

**Ventanas.** Para dos marcas consecutivas de la cola t(k-1) y t(k), la ventana k suma los bytes de los paquetes capturados en ese intervalo (B) y toma los descartes de la cola en el mismo intervalo (d). Se descartan los tramos donde el contador se reinicio, falto una muestra o hubo una pausa mayor de 2,5 s, y no se construyen filas que los crucen.

```
r_k = 8 * B_k / (10^6 * delta_k)      [Mbit/s]
d_k = D(t_k) - D(t_(k-1))
```

**Ejemplos.** Al cerrar la ventana k, las entradas son la tasa r(k), sus rezagos r(k-1) y r(k-2) y el numero de paquetes n(k). El objetivo es y(k) = d(k+1). Los rezagos se generan por corrida y por tramo continuo. Los contadores de la cola solo se usan para etiquetar y evaluar, y el calendario de tasas queda fuera de los predictores.

**Division.** Entrenamiento con c01, c02 y c03 (432 ejemplos), validacion con c04 (155) y prueba con c05 y c06 (208). Las corridas se mantienen completas y en orden de adquisicion.

**Modelo.** LinearRegression de scikit-learn, con las salidas negativas llevadas a cero:

```
y_k = -297.1 + 33.95*r_k + 14.51*r_(k-1) + 8.22*r_(k-2) + 0.16*n_k
```

La tasa y el numero de paquetes estan casi perfectamente relacionados, asi que los coeficientes no deben leerse uno a uno. Con una tasa sostenida, el modelo equivale aproximadamente a alertar cuando la tasa capturada pasa de unos 7,9 Mbit/s (calculo aproximado).

**Alerta.** Se emite al cerrar la ventana k si y(k) >= 275,4, y se compara con el evento d(k+1) >= 1. El umbral se eligio en la validacion maximizando la precision balanceada (promedio de sensibilidad y especificidad) y se fijo antes de la prueba.

**Referencias.** La regla simple alerta si la tasa capturada r(k) >= 5,67 Mbit/s, un umbral elegido en validacion con el mismo criterio. "Siempre alertar" sirve de piso.

La tasa capturada no pasa de 10,03 Mbit/s en ninguna de las seis corridas: la captura ve la salida ya limitada por la cola, no la carga ofrecida.

## 4. Resultados en prueba

En las 208 ventanas de prueba (c05 y c06), el modelo alerto con precision 0,897 y sensibilidad 0,830, con 9 falsas alarmas frente a las 34 de la regla simple, pero anuncio solo 2 de 12 inicios.

| Metodo | TN | FP | FN | TP | Precision | Sensibilidad | Precision balanceada |
|---|---|---|---|---|---|---|---|
| Regresion lineal (y >= 275,4) | 105 | 9 | 16 | 78 | 0,897 | 0,830 | 0,876 |
| Regla simple (tasa >= 5,67 Mbit/s) | 80 | 34 | 10 | 84 | 0,712 | 0,894 | 0,798 |
| Siempre alertar | 0 | 114 | 0 | 94 | 0,452 | 1,000 | 0,500 |

La precision balanceada se calculo a partir de la matriz de confusion. El error absoluto medio del modelo es de 150,9 descartes por ventana, frente a 201,8 de predecir siempre cero.

Un inicio observado es una ventana con d(k) = 0 y d(k+1) > 0, y se considera anunciado si hubo alerta al cerrar la ventana k.

| Corrida | Metodo | Precision | Sensibilidad | Inicios anunciados | Episodios en curso detectados |
|---|---|---|---|---|---|
| c05 | Regresion lineal | 0,955 | 0,808 | 0 de 4 | 21 de 22 |
| c05 | Regla simple | 0,815 | 0,846 | 0 de 4 | 22 de 22 |
| c06 | Regresion lineal | 0,877 | 0,838 | 2 de 8 | 55 de 60 |
| c06 | Regla simple | 0,681 | 0,912 | 4 de 8 | 58 de 60 |
| Total | Regresion lineal | 0,897 | 0,830 | 2 de 12 | 76 de 82 |
| Total | Regla simple | 0,712 | 0,894 | 4 de 12 | 80 de 82 |

Hay 12 inicios evaluables en total, pero solo 5 episodios planificados en estas dos corridas (3 en c05 y 2 en c06). Los demas inicios probablemente son descartes aislados con cargas de 8 a 10 Mbit/s o reapariciones dentro de un mismo episodio. Esto aun no se ha comprobado contra el calendario de fases.

## 5. Grafica temporal

Para cada corrida de prueba, el script del modelo genera una figura de tres paneles alineados en el tiempo:

- Panel superior: tasa capturada en Mbit/s, con el limite de 10 Mbit/s marcado.
- Panel central: descartes reales por ventana y descartes pronosticados, con cada pronostico ubicado en la ventana a la que se refiere (k+1), en escala logaritmica simetrica.
- Panel inferior: alertas (y >= umbral) e inicios observados, marcados con triangulos.

Corrida c05:

![Grafica c05](resultados/grafica_c05.png)

Corrida c06:

![Grafica c06](resultados/grafica_c06.png)

## 6. Discusion

El modelo reconoce sobre todo episodios que ya empezaron: anuncio 2 de 12 inicios, pero detecto 76 de las 82 ventanas de episodio en curso.

**Inicios frente a episodios en curso.** La regla simple anuncio 4 de 12 inicios y detecto 80 de 82 ventanas en curso. Ninguno de los dos metodos anticipa de forma fiable el comienzo de la congestion con las variables de la traza. La diferencia entre ellos esta en las falsas alarmas: 9 del modelo (1 en c05 y 8 en c06) frente a 34 de la regla (5 y 29).

**Rampas frente a saltos.** En c05, donde la carga salta de 5 a 14 Mbit/s, ni el modelo ni la regla anunciaron ninguno de los 4 inicios. En c06, con ascensos graduales, el modelo anuncio 2 de 8 y la regla 4 de 8, esta ultima con mas falsas alarmas. Las rampas dejan alguna senal en la tasa capturada; los saltos casi ninguna.

**Falsa alarma e inicio omitido.** El modelo omitio 10 inicios (4 en c05 y 6 en c06). Falta describir un caso concreto de cada tipo con la grafica: la ventana, la tasa capturada, el pronostico y que ocurrio despues.

**Que limita la prediccion.** La captura ve la salida de la cola, que no supera 10,03 Mbit/s, y no la carga que el emisor ofrece. Un salto de carga sin senales previas no deja huella en la tasa capturada ni en sus rezagos hasta que ya hay descartes, por lo que la prediccion a un intervalo vista se reduce a reconocer el episodio una vez iniciado. Esta explicacion es una interpretacion de los resultados y no se ha contrastado con el calendario de fases.

Con solo dos corridas de prueba y 12 inicios, una diferencia de dos inicios entre metodos no es concluyente.

## 7. Desviaciones y limitaciones

El montaje se aparta del enunciado en tres puntos, y el ensayo con la cola propuesta mostro que sus descartes a baja carga no eran congestion real.

**Cola.** Con burst de 16 kb y limite de 32 kb, una carga constante de 5 Mbit/s (la mitad del limite) produjo descartes en el 45 % de las ventanas, por rafagas del emisor. Se probo aumentar la cola con pruebas de 60 s (el 58 % de la primera fila a 8 Mbit/s sale de la corrida c01 con la cola original):

| Cola (burst / limite) | Ventanas con descarte a 5 Mbit/s | Ventanas con descarte a 8 Mbit/s |
|---|---|---|
| 16 kb / 32 kb (enunciado) | 45 % | 58 % |
| 16 kb / 64 kb | 25 % | 46 % |
| 32 kb / 128 kb | 0 % | 0 % |

Las corridas finales usan 32 kb / 128 kb. Las seis corridas con la cola del enunciado se conservan como comparacion.

**Buffer de envio.** Con la cola de 128 kb y la configuracion por defecto de iperf3, no hubo descartes ni a 14 Mbit/s: el emisor enviaba unos 9 Mbit/s en lugar de los 14 pedidos, probablemente porque su propio buffer de envio se llenaba antes de que se desbordara la cola. Con un buffer de 4 MB (`-w 4M`), a 14 Mbit/s hubo descartes en las 18 ventanas de la prueba y a 5 Mbit/s en 1 de 34.

**Enlace y equipos.** A es una maquina virtual de VirtualBox en un PC con Windows y B se conecto por WiFi al router, en lugar de dos equipos por Ethernet directo. En una prueba de 5 s a 8 Mbit/s, B reporto 12 % de datagramas perdidos mientras A no reportaba perdidas (capturas 3 y 4). No se contrasto con el contador de la cola en esa prueba, asi que no se sabe que parte fue del WiFi. La etiqueta de congestion usa solo los descartes de la cola de A.

**Ruido residual.** En las corridas finales todavia aparecen descartes a cargas bajas: entre 0 % y 15 % de las ventanas de 3 a 6 Mbit/s, y entre 23 % y 31 % de las de 8 a 9 Mbit/s. La hipotesis es que las pausas de la maquina virtual (el muestreo llego a tener pausas de 4,1 s en una prueba) provocan rafagas del emisor que desbordan la cola, pero no se ha comprobado.

**Tamano de la prueba.** Hay 869 ventanas en total, pero solo dos corridas de prueba, 12 inicios evaluables y una corrida de validacion. Las diferencias pequenas entre metodos no son concluyentes.

## Referencias

- Manual de iperf3 (ESnet)
- Manual de TBF (Linux, `man tc-tbf`)
- Manual de TShark (Wireshark)
