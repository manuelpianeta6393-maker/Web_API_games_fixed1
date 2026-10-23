# Respuestas del parcial

1. ¿Qué error encontraste en la validación de `stock`?

   El error estaba en la capa **Schema** (`schemas/game.py`). El campo `stock` de `GameCreate` usaba `Field(ge=0, le=50)`, así que aceptaba 0 unidades, aunque la regla exige un valor entre 1 y 50. Lo cambié a `Field(ge=1, le=50)`. Ahora Pydantic rechaza `stock: 0` con `422` antes de que la petición llegue al servicio.

2. ¿Qué hacía que un juego pareciera creado pero no quedara persistido?

   El error estaba en la capa **Service** (`create_game` en `services/game_service.py`). Se hacía `session.add(game)` y enseguida `session.refresh(game)`, sin `session.commit()`. `add` solo deja el objeto pendiente en la sesión: sin `commit` no se escribe en SQLite ni se genera el `id`, y el `refresh` falla o no recarga nada. Lo corregí con el orden visto en clase: `add` → `commit` → `refresh`. Así el registro queda guardado y el `id` generado por la base de datos se carga en el objeto.

3. ¿Qué problema tenía el endpoint que lista juegos?

   El error estaba en la capa **Router** (`routers/games.py`). `GET /games` declaraba `response_model=GameResponse`, que representa un solo objeto, pero el servicio devuelve una lista. FastAPI intentaba validar la lista como un único juego y fallaba. Lo cambié a `response_model=list[GameResponse]`, y ahora responde `200` con un arreglo JSON.

4. ¿Cómo solucionaste la consulta de un ID inexistente?

   El error estaba en la capa **Service** (`get_game`). `session.get(Game, game_id)` devuelve `None` cuando el ID no existe, y ese `None` se devolvía sin control, lo que producía un error de validación en lugar de un `404`. Agregué la verificación `if not game: raise HTTPException(status_code=404, detail="Game not found")`, igual que en `delete_game`. Ahora la respuesta es `404` con `{"detail": "Game not found"}`.

5. ¿Por qué DELETE podía responder correctamente y aun así no eliminar el registro de SQLite?

   El error estaba en la capa **Service** (`delete_game`). Se ejecutaba `session.delete(game)` y se devolvía el mensaje de éxito sin `session.commit()`. `delete` solo marca el objeto para eliminarlo dentro de la sesión. Al cerrarse la sesión sin `commit`, la transacción se descarta y el registro sigue en SQLite. Agregué `session.commit()` después de `session.delete(game)`, y ahora la eliminación es persistente: después de reiniciar el servidor, el juego borrado no vuelve a aparecer.

## Verificación

| Prueba | Resultado |
|---|---|
| P01 Crear juego válido | 201, `id` generado, `active: true` |
| P02 Validar stock = 0 | 422 |
| P03 Listar juegos | 200, lista |
| P04 Consultar existente | 200 |
| P05 Consultar inexistente | 404, `Game not found` |
| P06 Eliminar | 200, `Game deleted`; luego GET → 404 |
| P07 Persistencia tras reiniciar | Se conservan los no eliminados; los eliminados no reaparecen |
