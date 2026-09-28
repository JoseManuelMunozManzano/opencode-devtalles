---
name: pokemon-info
description: Use when user mentions Pokémon, Pokemon, Pokedex, Pokédex, or asks about any Pokémon by name or ID, stats, types, abilities, moves, height, weight, sprites, evolutions, or compares Pokémon. Fetches live data from PokeAPI.
---

# Pokémon Info

Consulta PokeAPI para responder cualquier tema sobre Pokémon con datos actuales. Nunca inventar datos. Sin llamada API, sin respuesta definitiva.

## Cuándo usar

- Mencionan `Pokémon`, `Pokemon`, `Pokédex`, `Pokedex`.
- Preguntan por nombre (`ditto`, `pikachu`, `charizard`) o ID (`1`, `25`, `132`).
- Piden tipos, stats, habilidades, movimientos, altura, peso, experiencia base, sprites, gritos, especie, evoluciones.
- Comparan dos o más Pokémon.
- Hablan de formas regionales, mega, gmax, variantes con guiones (`mr-mime`, `ho-oh`, `tapu-koko`, `mime-jr`).

## Fuente única

Base:

```
https://pokeapi.co/api/v2/pokemon/{identifier}
```

Ejemplos válidos:

```
https://pokeapi.co/api/v2/pokemon/ditto
https://pokeapi.co/api/v2/pokemon/1
https://pokeapi.co/api/v2/pokemon/pikachu
https://pokeapi.co/api/v2/pokemon/25
```

- Sin API key. GET público.
- Herramienta: `webfetch` con URL completa.
- Un request por Pokémon. Comparación de N Pokémon = N requests.

## Normalizar identificador

1. Minúsculas, recortar espacios: ` Pikachu ` -> `pikachu`.
2. Espacios, puntos, apóstrofes -> guion o eliminar según PokeAPI:
   - `Mr Mime` -> `mr-mime`
   - `Mr. Mime` -> `mr-mime`
   - `Farfetch'd` -> `farfetchd`
   - `Ho-Oh` -> `ho-oh`
   - `Tapu Koko` -> `tapu-koko`
   - `Nidoran♀` -> `nidoran-f`, `Nidoran♂` -> `nidoran-m`
3. IDs: usar número directo sin ceros extra: `001` -> `1`.
4. Si falla con nombre normalizado, probar variantes con sufijo regional (`-alola`, `-galar`, `-hisui`, `-paldea`) antes de declarar no encontrado.

## Procedimiento

1. Detectar todos los Pokémon nombrados en mensaje.
2. Construir URL por cada uno.
3. Llamar `webfetch` (formato `text` o `markdown`; JSON viene como texto).
4. Si status 404 / "Not Found": detenerse para ese Pokémon, informar nombre no válido, sugerir verificar ortografía o ID 1-1025.
5. Si fallo red / timeout: reintentar una vez. Si persiste, decirlo, no inventar.
6. Extraer solo campos necesarios de respuesta. Respuesta completa es enorme (sprites, moves, game_indices); no pegarla entera al usuario.
7. Si pregunta requiere más contexto (descripción, hábitat, cadena evolutiva):
   - Leer `species.url` (ej: `https://pokeapi.co/api/v2/pokemon-species/132/`).
   - Desde especie, leer `evolution_chain.url` si piden evoluciones.
   - Para efecto de habilidad/movimiento, seguir `ability.url` o `move.url` solo si usuario lo pide.

## Campos clave del endpoint `/pokemon/{id|name}`

- `id`: número Pokédex.
- `name`: nombre normalizado.
- `types[].type.name`: tipos (`slot` define orden).
- `abilities[]`: `ability.name` + `is_hidden`.
- `stats[]`: `stat.name` + `base_stat` (`hp`, `attack`, `defense`, `special-attack`, `special-defense`, `speed`).
- `height`: decímetros. Convertir: `m = height / 10`.
- `weight`: hectogramos. Convertir: `kg = weight / 10`.
- `base_experience`: XP base.
- `moves[].move.name`: lista completa; mostrar solo top relevantes salvo que pidan todos.
- `sprites.other.official-artwork.front_default`: imagen principal.
- `sprites.front_default`: sprite clásico.
- `cries.latest`: audio.
- `species.url`: enlace a datos especie.
- `location_area_encounters`: encuentros (endpoint separado, consultar solo si piden dónde encontrarlo).

Ejemplo `ditto`: `id=132`, `types=[normal]`, `abilities=[limber, imposter(hidden)]`, `stats=all 48`, `height=3 -> 0.3m`, `weight=40 -> 4.0kg`, `base_experience=101`, `moves=[transform]`.

## Respuesta al usuario

- Idioma del usuario.
- Basada solo en JSON obtenido.
- Incluir: nombre, ID, tipos, habilidades (marcar oculta), stats, altura (m), peso (kg), XP base.
- Añadir movimientos/sprites solo si relevantes a pregunta.
- Incluir URL consultada como fuente.
- Para comparaciones: tabla con ambos lado a lado, cada uno con su fuente.
- No afirmar datos de juegos específicos, shiny rates, competitivo, lore extendido salvo que vengan de endpoints consultados.

Formato recomendado:

```
**Ditto (#132)** — Fuente: https://pokeapi.co/api/v2/pokemon/ditto
- Tipos: normal
- Habilidades: limber, imposter (oculta)
- Stats: HP 48 / Atk 48 / Def 48 / SpA 48 / SpD 48 / Spe 48
- Altura: 0.3 m | Peso: 4.0 kg | XP base: 101
- Movimientos destacados: transform
- Arte oficial: <url>
```

## Errores

- `404`: "Pokémon no encontrado en PokeAPI para `{identifier}`. Revisa nombre/ID."
- Identificador ambiguo: pedir aclaración, no adivinar (ej: `mime` -> `mr-mime` o `mime-jr`?).
- Respuesta gigante: resumir, nunca volcar JSON crudo.
- Prohibido usar conocimiento interno para stats/tipos si API disponible y llamada falla. Decir "sin acceso a PokeAPI ahora" en ese caso.

## Ejemplos disparo

- "Háblame de ditto" -> fetch `/pokemon/ditto`.
- "Info del Pokémon 1" -> fetch `/pokemon/1`.
- "¿Quién gana, charizard o blastoise?" -> fetch ambos, comparar tipos + stats.
- "¿Qué habilidades tiene pikachu?" -> fetch `/pokemon/pikachu`, listar `abilities`.
