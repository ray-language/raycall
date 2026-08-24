# raycall

Microservicios de demostración **conectados de verdad**, escritos en [raylang](https://github.com/roberto-ayala/raylang): un front HTTP (`net/webserver`) delante de dos servicios `packages/rpc` (orders → inventory), con el arco M88 completo en acción — **trace W3C propagado de punta a punta** (un span hijo por salto), **deadlines en cascada** (el presupuesto del cliente se recorta en cada hop), logs JSON estampados con el `trace_id` entrante, y códigos de estado honestos (409 sin stock, 504 deadline, 502 upstream).

```text
$ raycall demo
$ curl -s -X POST localhost:7480/orders \
    -H 'traceparent: 00-cafecafe…-1234…-01' -d '{"sku":"tornillo","qty":5}'
{"order_id":"01a035d0-…","trace_id":"cafecafe…",
 "orders_saw_trace":"00-cafecafe…-11b4f3e0…-01",
 "inventory_saw_trace":"00-cafecafe…-3c1d8824…-01", …}
```

La respuesta enseña la cadena entera: el MISMO `trace_id` en los tres
procesos y un `span_id` distinto por salto — la propagación se ve con un
curl, sin colector.

## Qué demuestra

- **Deadline en cascada**: `X-Deadline-Ms: 250` en el front → orders recibe
  el presupuesto en `rpc.Req.deadline_ms`, mide su propio gasto con
  `resilience.deadline` y pasa SOLO el resto a inventory; un almacén que
  tarda 800 ms muere en el salto correcto y el front responde 504 (test).
- **Trace por salto**: `webserver.trace_of` adopta el traceparent entrante,
  cada `rpc.call_full` viaja con un span HIJO, y `net/log.with_trace` estampa
  cada línea. El test verifica el mismo trace_id en los tres servicios y
  spans distintos.
- **Estado en actor** (inventory), errores como valores de punta a punta
  (`Err` del handler RPC → `Err` del cliente → status HTTP), apagado graceful
  en los tres.

Los tres servicios corren como procesos separados (`raycall inventory|orders|front`)
o juntos en uno (`raycall demo`).

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| front HTTP → orders RPC → inventory RPC, E2E | ✅ |
| Trace W3C propagado (verificado en los 3 saltos) | ✅ |
| Deadline en cascada (504 en el salto correcto) | ✅ |
| Códigos honestos: 409/502/504 + logs JSON con trace_id | ✅ |
| Binario nativo (demo verificada con curl) | ✅ |
| Tests (E2E completo con 3 servicios reales) | ✅ 1 |
| Pata gRPC real (`grpc_client` contra un servicio externo) | 📋 v2 — sin dogfood aún |
| Métricas por servicio, retry entre servicios | 📋 v2 |

## Hallazgos de dogfood

Anotados en `raylang/IDEAS.md` §72:

1. **El arco M88 compone entero y a la primera**: rpc (deadline_ms +
   traceparent en el frame) + trace (child por salto) + log (with_trace) +
   resilience (deadline) encajan sin fricción entre tres procesos.
2. **Conexión RPC por petición**: el cliente rpc es secuencial por conexión;
   handlers concurrentes no pueden compartir uno (se desincroniza) → cada
   llamada conecta/desconecta. Un pool o multiplexación por id es el hueco
   de producción de `packages/rpc` (su README ya insinúa streaming diferido).
3. `grpc_client` queda como la última superficie de red sin dogfood (necesita
   un servicio gRPC externo real).

## Desarrollo

```sh
ray test
ray run src/main.ray demo
ray build --native src/main.ray -o raycall --release
```

Estructura: `src/main.ray` (CLI + boot) · `inventory.ray` (actor + RPC) ·
`orders.ray` (cascada de deadline + child spans) · `front.ray` (HTTP + trace
+ status codes).
