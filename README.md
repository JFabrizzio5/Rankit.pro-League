# Ecosistema Rankit.Pro League - Plan de Integración de API y Tratamiento de Datos

Este documento detalla el plan técnico para la migración de nuestro simulador local hacia una integración real con la API de Rankit.pro. Aquí se explica la arquitectura de endpoints, el modelo de datos de los competidores y los algoritmos que determinan las clasificaciones directas, la tabla de constancia general y la lógica de exclusión del repechaje.

---

## 1. Arquitectura de Endpoints (API Mock)

Para conectar el Dashboard de la liga, consumiremos los siguientes recursos desde la API de Rankit.pro:

### A. Obtener Estado y Configuración de Fases
* **Endpoint**: `GET /api/v1/league/phases`
* **Descripción**: Devuelve los detalles de las 5 fases (Clasificatorio 1, Clasificatorio 2, Repechaje, Gran Final, Tabla de Constancia), sus estados actuales ("Aún no se juega", "En Vivo", "Finalizado") y configuraciones de cupos.

### B. Obtener Roster de Participantes / Leaderboard de una Fase
* **Endpoint**: `GET /api/v1/league/phases/{phase_id}/leaderboard`
* **Descripción**: Devuelve la lista ordenada de jugadores con sus estadísticas acumuladas (puntos, eliminaciones, victorias magistrales, partidas disputadas).

### C. Procesar Registro de Jugador
* **Endpoint**: `POST /api/v1/league/phases/{phase_id}/register`
* **Cuerpo de la Petición**:
```json
{
  "epic_id": "Ninja_On_Rankit",
  "email": "jugador@correo.com",
  "whatsapp": "+525512345678",
  "discord": "Gamer#1337",
  "follow_verified": true,
  "rankit_account_verified": true
}
```
* **Validación**: La API comprueba que el `email` esté registrado previamente en Rankit.pro y que las credenciales sociales sean válidas.

---

## 2. Modelos de Datos en el Frontend

Trataremos los datos obtenidos en las siguientes estructuras de objetos JavaScript:

```typescript
interface PlayerStandings {
  name: string;        // Epic ID del competidor
  points: number;      // Puntos totales acumulados
  kills: number;       // Eliminaciones totales
  vr: number;          // Victorias Magistrales (Victory Royale)
  matches: number;     // Cantidad de partidas disputadas
  avatar: string;      // Icono representativo del jugador
  rank?: number;       // Posición actual calculada
  qualified?: boolean; // Bandera de clasificación directa
  qualificationType?: 'Direct' | 'Repechaje' | 'None';
}

interface ConstancyPlayer {
  name: string;
  points: number;      // Puntos Semana 1 + Puntos Semana 2
  w1_pts: number;      // Puntos específicos en Fase 1
  w2_pts: number;      // Puntos específicos en Fase 2
  matches: number;
  avatar: string;
  rank?: number;
  status: 'GOLD' | 'RED' | 'NORMAL'; // GOLD = Clasificado final, RED = Clasificado repechaje
}
```

---

## 3. Algoritmos de Tratamiento y Reglas de Negocio

El procesamiento de datos en el cliente (o servidor intermedio) sigue reglas competitivas estrictas basadas en el formato Solos de Fortnite:

### A. Regla del 15% (Semana 1 y Semana 2)
1. Al finalizar las partidas de la Semana 1 o Semana 2, se ordena a los competidores por `puntos` (descendente), desempatando por `kills` y luego por `vr`.
2. Se calcula el número de pases directos:
   $$\text{Clasificados} = \lceil \text{Total Participantes} \times 0.15 \rceil$$
3. Los jugadores en este rango se marcan como clasificados directos y sus IDs se añaden al conjunto global de clasificados:
   `const globalFinalists = new Set<string>();`
4. **Regla de Exclusión**: Estos jugadores quedan bloqueados y **no son elegibles para participar en el Repechaje**, permitiendo que los cupos de constancia y repechaje se desplacen a jugadores que no han clasificado.

### B. Algoritmo de la Tabla de Constancia (W1 + W2)
Para construir la pestaña de Constancia General, unimos los resultados de las dos primeras semanas:
1. Extraemos los rosters de la Fase 1 y la Fase 2.
2. Agrupamos los datos por `Epic ID` sumando los puntos y partidas de ambas semanas.
3. Clasificamos a los jugadores visualmente según su estado:
   * **Dorado (GOLD)**: Si el Epic ID del jugador ya se encuentra en la lista de clasificados directos (Top 15% de W1 o W2). Estos jugadores se muestran con corona dorada y la etiqueta `CLASIFICADO`.
   * **Rojo (RED)**: Excluyendo a los jugadores dorados (ya clasificados), tomamos a los **Top 20 jugadores** con mayor puntaje acumulado. Estos 20 jugadores obtienen su pase directo para competir en el **Repechaje** de la Semana 3.
   * **Estándar (NORMAL)**: Jugadores restantes que no clasificaron a la final ni alcanzaron cupo de repechaje.

### C. Lógica del Repechaje (Semana 3 - Muerte Súbita)
El Repechaje consta de **4 partidas independientes**. Para evitar que un jugador clasifique dos veces y asegurar la rotación de cupos:
1. **Exclusión en Tiempo Real**: Cuando un jugador obtiene una Victoria Magistral (gana la partida), se clasifica automáticamente a la Gran Final.
2. **Marcador de Estado**: Su Epic ID es coloreado en **ROJO** con la etiqueta `GANADOR M[X] - CLASIFICADO`.
3. **Bloqueo de Puntuación**: Para las partidas restantes del repechaje, este jugador no acumula más puntos ni puede ganar otra partida (simulado omitiendo su participación en los lobbies de las partidas subsecuentes).
4. **Reparto de Pases Finales (8 Pases)**:
   * **4 Ganadores**: El ganador de cada una de las 4 partidas del repechaje (excluyendo ganadores previos).
   * **3 Constancia**: Los 3 jugadores con mayor cantidad de puntos totales acumulados en las 4 partidas (excluyendo a los 4 ganadores).
   * **1 MVP Kills**: El jugador con más kills acumuladas en el Repechaje que no pertenezca al grupo de los 4 ganadores ni a los 3 de constancia.

---

## 4. Visualización en la Interfaz (CSS / Tailwind)

Para reflejar el estado competitivo en la tabla de clasificación, se utilizan los siguientes estilos:

| Color | Estado Competitivo | Aplicación Visual |
|---|---|---|
| **Dorado (`#f1c40f`)** | Ya clasificado a la Gran Final (Top 15% W1/W2) | Fila de tabla con bordes/sombras doradas, corona dorada y badge `FINAL`. |
| **Rojo (`#d7263d`)** | Elegible para el Repechaje (Top 20 Constancia) o Ganador de partida en Repechaje | Fila/Badge destacado en rojo brillante, texto `REP` o `CLASIFICADO`. |
| **Neutral (`#gray`)** | Aún en competencia o no clasificado | Fila con estilos estándar oscuros y semitransparentes. |

---

## 5. Simulación de Flujo Local vs API

Cuando la API esté lista, reemplazaremos las funciones locales `simulateMatch()` y `calculateConstancyStandings()` por llamadas `fetch()` asíncronas:

```javascript
async function loadPhaseLeaderboard(phaseId) {
    try {
        const response = await fetch(`/api/v1/league/phases/${phaseId}/leaderboard`);
        const data = await response.json();
        renderLeaderboard(data.players);
    } catch (error) {
        showToast("Error de Servidor", "No se pudieron obtener los datos de la liga.", "exclamation-triangle");
    }
}
```