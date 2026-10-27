# StarCraft II LAN: investigación técnica

Estado: investigación en curso · última actualización: 2026-10-27

## Objetivo

Evaluar si StarCraft II 5.0.17.98310 puede ofrecer salas y partidas entre dos
PC de una LAN sin conexión a Internet ni servicios oficiales de Battle.net,
conservando el cliente y la interfaz oficiales. El juego se inicia mediante
`Support64/SC2Switcher_x64.exe`, que prepara y lanza `SC2_x64.exe` con sus
parámetros.

Este repositorio contiene notas y referencias. No incluye archivos del juego,
ejecutables modificados, volcados de memoria, certificados privados, claves,
tokens ni capturas crudas.

## Resumen del estado

| Tema | Estado | Evidencia / límite |
|---|---|---|
| HostSpy funciona como proxy para tráfico HTTP/HTTPS del cliente Battle.net | Observado en las pruebas del proyecto | No implica que SC2 acepte el mismo certificado ni que el proxy funcione como servidor BGS. |
| `SC2_x64.exe` usa el flujo BGS por TCP 1119 | Observado | El protocolo de aplicación de esa conexión todavía no se ha recibido con éxito en el cliente SC2. |
| En un ensayo, TLS con HostSpy y con el upstream terminó; SC2 cerró antes de enviar datos de aplicación | Observado en una captura | No identifica por sí solo la causa del cierre. |
| SC2 rechaza el certificado por pinning SPKI | Hipótesis | No confirmado. También pueden intervenir el bundle, otra validación, una condición de sesión o el arranque BGS. |
| Se sirvió un bundle personalizado y SC2 lo aceptó | No confirmado | Los logs de un intento posterior no muestran una solicitud del bundle ni tráfico TLS a 1119. |
| Un parche de laboratorio permite arrancar normalmente | No | El intento registrado produjo una violación de acceso antes de llegar al flujo BGS. |
| El protocolo BGS/Aurora local permite crear una sala y jugar entre dos clientes | Pendiente | Aún no se ha completado el inicio de sesión BGS de SC2 contra un servidor local. |

## Evidencia y alcance

### Red y TLS

- HostSpy redirige nombres configurados mediante el archivo `hosts`; puede
  terminar TLS para observar aplicación o tunelar TLS extremo a extremo.
- TCP 1119 está asociado al flujo BGS. El WebSocket `v1.rpc.battle.net` y el
  framing protobuf se observaron en una captura del cliente de escritorio de
  Battle.net (`BGS C++ SDK 8.1.0`), no se deben atribuir automáticamente a SC2.
- Una negociación TLS completada no demuestra que la capa de aplicación haya
  aceptado el certificado, el bundle o la identidad del servidor.
- La petición investigada del bundle es
  `nydus.battle.net/Bnet/zxx/client/bgs-key-fingerprint`. También se observó la
  ruta `keys.actual.battle.net/pki/prod/fingerprints.bkp`. El cuerpo y la
  respuesta deben coincidir con el formato esperado; el `Content-Type`
  configurado para el override debe revisarse junto con los bytes servidos.
- El bundle usa un cuerpo JSON seguido por un tráiler `NGIS` y una firma RSA.
  Cambiar el JSON sin producir una firma correspondiente invalida la firma.
  La validez criptográfica offline de un bundle de laboratorio no demuestra que
  el cliente lo acepte durante su arranque.

### Bundle y binario de laboratorio

- Un ensayo modificó únicamente el módulo RSA embebido que verifica la firma
  del bundle: 256 bytes en el offset de archivo `0x3267850`.
- El bundle local verificó offline con la clave privada de laboratorio. Esa
  comprobación solo demuestra que la firma y la clave emparejan; no prueba que
  la ruta de carga sea correcta ni que el cliente lo acepte.
- El cliente produjo `ACCESS_VIOLATION` al leer `0x8`; el RIP del dump fue
  `0x7FF65EFA6BE0` y el contexto mostraba `RCX = 0`. En ese intento HostSpy no
  registró una petición del bundle ni una conexión TLS a 1119.
- El fallo se correlacionó con la ejecución del binario de laboratorio, pero
  todavía no se aisló la causa. No se debe presentar como una prueba de que el
  parche RSA sea el origen exacto ni como evidencia de pinning.
- El ejecutable retail se restauró y se verificó contra su hash original en el
  equipo de pruebas. Ninguna copia del ejecutable se publica aquí.

### Estado operativo de las herramientas

- [**HostSpy**](https://github.com/Chichimokers/hostspy) es un proxy e
  interceptor de transporte. No implementa por sí solo el login BGS/Aurora ni
  las salas de SC2.
- [**NetTrace**](https://github.com/Chichimokers/nettrace) observa procesos,
  endpoints TCP/UDP y consultas DNS. No descifra TLS ni captura bytes de
  aplicación.
- El modo passthrough de HostSpy permite observar que una conexión ocurre,
  pero mantiene cifrados los mensajes TLS.
- Los logs de aplicación y WebSocket pueden contener correo, BattleTag, IDs,
  sesiones, claves o tokens. No publicar capturas crudas ni volcados de cliente.

## Revisión de proyectos relacionados

### StarFriend

StarFriend es un antecedente de LAN con interfaz de SC2. La guía archivada de
2011 pide un cliente taiwanés 1.3.2, inicia un servidor StarFriend, configura
la IP del host y usa el flujo de lobby del juego. Es una solución acoplada a
clientes antiguos; no encontramos una fuente mantenida que demuestre
compatibilidad con 5.0.17.

- [Guía de instalación de StarFriend](https://www.darkblizz.org/Forum2/index.php?topic=2333.0)
- [Anuncio de StarFriend 0.43, cliente 1.3.2](https://www.darkblizz.org/Forum2/index.php?topic=2281.0%3Bprev_next%3Dnext)
- [Informe comunitario de StarFriend 1.54 y SC2 2.0.9](https://www.reddit.com/r/starcraft/comments/ketfab/anyone_still_using_starfriend/)

Sirve como referencia histórica para el modelo host/cliente y lobby, no como
componente directamente reutilizable para el ejecutable actual.

### Experimental BGS Emulator

Se revisó [ProjectHiNAtyu/Experimental-BGS-Emulator](https://github.com/ProjectHiNAtyu/Experimental-BGS-Emulator), una prueba de concepto en C++ para Windows. Incluye routing Winsock en proceso, un endpoint TLS con SChannel, WebSocket y respuestas Aurora/protobuf de conexión, login, cuenta y sesión, además de algunos recursos locales.

Es el punto de partida público más próximo al flujo actual, pero **no es un
servidor listo para SC2 5.0.17**:

- El README deja la integración específica del cliente (certificado, SPKI,
  bundle, nombre del host e identidad) al proyecto anfitrión.
- Enumera `SC2 ` como etiqueta de programa, y aclara que eso no demuestra
  compatibilidad con un título o build.
- El código actual devuelve un bundle de huellas vacío y no incluye los hooks
  específicos de validación del cliente SC2.
- La DLL debe cargarse en el proceso del juego antes de que empiece a conectarse;
  el repositorio no proporciona una integración lista con `SC2Switcher`.
- Sus stubs de validación son ejemplos. El propio README advierte que ABI,
  argumentos y valores de retorno varían, y que aceptar el bundle puede
  requerir conservar estado interno del SDK.

La copia consultada para análisis local está fuera de este repo. Para usar este
proyecto habría que implementar y validar el host/loader de SC2, identificar
sus hooks reales y comprobar las RPC y recursos requeridos por esta build.

### Otros repositorios

- [Blizzard/s2client-proto](https://github.com/Blizzard/s2client-proto)
  documenta la API del cliente de SC2 (bots y control del juego); no implementa
  Battle.net BGS/Aurora.
- `ageLANServer` de este workspace implementa lobbies para juegos de Age of
  Empires/Age of Mythology, no para SC2.

## Camino técnico propuesto

1. **Fijar una referencia estable:** iniciar el ejecutable retail restaurado
   con `SC2Switcher_x64.exe` y registrar si llega al menú sin el crash del
   binario de laboratorio.
2. **Capturar desde el arranque:** correlacionar hora, PID/árbol de procesos,
   DNS y endpoints con NetTrace y los logs de HostSpy. Distinguir si SC2 pide
   el bundle, llega a 1119 y completa TLS. No asumir pinning a partir del FIN.
3. **Localizar el flujo en la build exacta:** en IDA, seguir strings y
   referencias del bundle, hostname y errores criptográficos; confirmar
   direcciones, argumentos, valores de éxito y estado esperado en el llamador.
4. **Probar una integración aislada y reversible:** evitar modificar el retail;
   cargar una DLL de laboratorio en el proceso correcto y limitar cualquier
   aceptación de identidad al endpoint/certificado local de la prueba.
5. **Completar el emulador:** después de superar el arranque TLS/BGS, responder
   las RPC, recursos, identidad, licencia y sesión que esta build realmente
   solicita. No usar los valores placeholder como si fueran universales.
6. **Validar dos clientes:** login de ambos, creación/unión al lobby, carga de
   mapa, inicio de partida y tráfico de juego en la LAN sin Internet.

El paso 1 es prioritario: el ejecutable retail está restaurado en el equipo de
pruebas. Si también falla sin el parche, primero hay que resolver el problema
de arranque y no atribuirlo a BGS.

## Referencias

- [Experimental BGS Emulator — README y código](https://github.com/ProjectHiNAtyu/Experimental-BGS-Emulator)
- [StarFriend — guía archivada](https://www.darkblizz.org/Forum2/index.php?topic=2333.0)
- [StarFriend 0.43 — requisitos de cliente](https://www.darkblizz.org/Forum2/index.php?topic=2281.0%3Bprev_next%3Dnext)
- [Blizzard s2client-proto](https://github.com/Blizzard/s2client-proto)
- [Paper: The Revolution of StarCraft Network Traffic](https://myslu.stlawu.edu/~clee/docs/starcraft2.netgames.2012.pdf)

## Privacidad y archivos excluidos

No incluir en issues, commits o pull requests: ejecutables del juego, dumps,
logs HTTP/TLS/WebSocket, bases de NetTrace, certificados y claves privadas,
bundles firmados localmente, tokens, credenciales, IDs de cuenta o BattleTags.
Los logs descifrados pueden contener datos de sesión incluso cuando las
cabeceras HTTP se redactan.
