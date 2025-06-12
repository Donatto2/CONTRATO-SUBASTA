# Contrato Inteligente de Subasta

Este repositorio contiene el código fuente y la documentación de un contrato inteligente de subasta desarrollado en Solidity. Este contrato implementa un sistema de subasta transparente con reembolsos automáticos para los postores no ganadores, incorporando características como extensiones de tiempo en ofertas tardías, incrementos mínimos de oferta, y un manejo seguro de fondos.

## Tabla de Contenidos

- [Funcionalidades Clave](#funcionalidades-clave)
- [Estructura del Contrato](#estructura-del-contrato)
  - [Variables de Estado](#variables-de-estado)
  - [Estructuras](#estructuras)
  - [Eventos](#eventos)
  - [Modificadores](#modificadores)
  - [Constructor](#constructor)
  - [Funciones Principales](#funciones-principales)
  - [Funciones de Consulta (View Functions)](#funciones-de-consulta-view-functions)
- [Consideraciones de Seguridad](#consideraciones-de-seguridad)
- [Despliegue y Uso](#despliegue-y-uso)
- [Licencia](#licencia)

## Funcionalidades Clave

El contrato de subasta implementa las siguientes funcionalidades clave:

*   **Extensiones de Tiempo en Ofertas Tardías:** Si una oferta se realiza cerca del final de la subasta, el plazo se extiende automáticamente para permitir una competencia justa.
*   **Incrementos Mínimos de Oferta:** Las nuevas ofertas deben superar la oferta actual en al menos un 5%.
*   **Comisión del 2%:** Se aplica una comisión del 2% sobre las ofertas ganadoras y los reembolsos a los postores no ganadores.
*   **Patrones de Retiro Seguros:** Implementa mecanismos seguros para el retiro de fondos.
*   **Reembolsos Automáticos:** Los postores no ganadores reciben un reembolso automático de sus fondos (menos la comisión).
*   **Retiro de Exceso de Fondos:** Los postores pueden retirar el monto de su oferta que excede el mínimo requerido para mantener su posición, si su oferta ya no es la más alta.

## Estructura del Contrato

El contrato `Auction.sol` está estructurado de la siguiente manera:

### Variables de Estado

Las variables de estado almacenan la información persistente del contrato en la blockchain. A continuación, se detallan las variables más relevantes:

```solidity
    address private owner;
    uint private deadline;
    uint private constant commissionPercent = 2; // 2% commission
    uint private constant timeExtension = 10 minutes;
    bool private ended;
    Bid[] private bidHistory;
    Bid private highestBid;
    mapping(address => uint) private accumulatedBids;
    mapping(address => uint[]) private userBidHistory;
    mapping(address => uint) public pendingWithdrawals;
```

*   `owner`: La dirección del propietario del contrato, quien tiene privilegios administrativos.
*   `deadline`: Marca de tiempo (timestamp) que indica cuándo finaliza el período de puja de la subasta.
*   `commissionPercent`: Una constante que define el porcentaje de comisión que se toma sobre las ofertas (2% en este caso).
*   `timeExtension`: Una constante que define la duración en segundos (10 minutos) que se añade a la fecha límite de la subasta si se realiza una oferta en el último momento.
*   `ended`: Un indicador booleano que es `true` una vez que la subasta ha finalizado, impidiendo cambios de estado adicionales.
*   `bidHistory`: Un array que almacena el historial completo de todas las ofertas válidas recibidas, en orden cronológico.
*   `highestBid`: Una estructura `Bid` que representa la oferta más alta válida actual en la subasta.
*   `accumulatedBids`: Un mapeo que rastrea el monto total acumulado de ofertas por cada dirección de postor.
*   `userBidHistory`: Un mapeo que almacena el historial de todos los montos de ofertas realizadas por cada dirección de usuario.
*   `pendingWithdrawals`: Un mapeo que rastrea los montos reembolsables (98% de las ofertas) por cada dirección de postor.

### Estructuras

Las estructuras (`structs`) permiten definir tipos de datos personalizados y agrupar variables relacionadas. En este contrato, se utiliza la siguiente estructura:

```solidity
    struct Bid {
        address bidder;
        uint amount;
    }
```

*   `Bid`: Representa una única oferta en el sistema de subasta, conteniendo:
    *   `bidder`: La dirección del postor que realizó esta oferta.
    *   `amount`: El monto de ETH ofertado (en wei).

### Eventos

Los eventos son una forma de registrar acciones en la blockchain, permitiendo que las aplicaciones externas (como interfaces de usuario o servicios de monitoreo) reaccionen a los cambios de estado del contrato. Los eventos definidos en este contrato son:

```solidity
    event NewBid(address indexed bidder, uint amount);
    event ExcessFundsAvailable(address indexed bidder, uint amount);
    event AuctionEnded(address winner, uint winningAmount);
    event PartialRefund(address indexed bidder, uint amount);
    event DepositWithdrawn(address indexed user, uint refundAmount, uint fee);
    event EmergencyWithdraw(address indexed receiver, uint amount);
    event DepositRefunded(address indexed bidder, uint refundAmount, uint fee);
```

*   `NewBid`: Emitido cuando se realiza una nueva oferta. Contiene la dirección del postor y el monto de la oferta.
*   `ExcessFundsAvailable`: Emitido cuando un postor realiza una oferta que crea fondos en exceso, disponibles para retiro.
*   `AuctionEnded`: Emitido cuando la subasta concluye. Informa sobre el ganador y el monto de la oferta ganadora.
*   `PartialRefund`: Emitido cuando un postor retira fondos en exceso de su oferta.
*   `DepositWithdrawn`: Emitido cuando un usuario retira sus depósitos (reembolsos).
*   `EmergencyWithdraw`: Emitido cuando los fondos del contrato son retirados bajo una situación de emergencia por el propietario.
*   `DepositRefunded`: Emitido cuando las ofertas no ganadoras son reembolsadas automáticamente.

### Modificadores

Los modificadores son fragmentos de código que se pueden reutilizar para validar condiciones antes de ejecutar una función. Esto mejora la legibilidad y reduce la duplicación de código. Los modificadores utilizados son:

```solidity
    modifier onlyBeforeEnd() {
        require(block.timestamp < deadline, "Auction has ended");
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
```

*   `onlyBeforeEnd()`: Restringe la ejecución de la función solo si la subasta aún no ha terminado.
*   `onlyOwner()`: Restringe la ejecución de la función solo al propietario del contrato.

### Constructor

El constructor es una función especial que se ejecuta una única vez cuando el contrato es desplegado en la blockchain. Se encarga de inicializar las variables de estado esenciales para el funcionamiento de la subasta.

```solidity
    constructor(uint _durationSeconds) {
        owner = msg.sender;
        deadline = block.timestamp + _durationSeconds;
    }
```

El constructor recibe un parámetro:

*   `_durationSeconds`: La duración de la subasta en segundos desde el momento del despliegue.

Inicializa el propietario del contrato y calcula la fecha límite de la subasta.

### Funciones Principales

Estas funciones son el núcleo de la lógica de la subasta, permitiendo a los usuarios interactuar con el contrato para realizar ofertas, finalizar la subasta y gestionar sus fondos.

```solidity
    function placeBid() external payable onlyBeforeEnd {
        // ... (código de la función)
    }

    function endAuction() external onlyOwner {
        // ... (código de la función)
    }

    function claimRefund() external returns (bool success) {
        // ... (código de la función)
    }

    function withdrawExcess() external onlyBeforeEnd returns (bool success) {
        // ... (código de la función)
    }

    function emergencyWithdraw() external onlyOwner {
        // ... (código de la función)
    }
```

*   `placeBid()`: Permite a cualquier participante enviar Ether para realizar una oferta. Valida que la oferta cumpla con el incremento mínimo (5% más que la mejor oferta actual) y extiende la subasta si se realiza en los últimos 10 minutos. Emite `NewBid` y `ExcessFundsAvailable` si aplica.
*   `endAuction()`: Permite al propietario finalizar la subasta una vez que ha pasado la fecha límite. Marca la subasta como finalizada, transfiere la oferta ganadora (menos el 2% de comisión) al propietario, y automáticamente reembolsa el 98% a los postores no ganadores. Emite `AuctionEnded` y `DepositRefunded`.
*   `claimRefund()`: Permite a los postores retirar sus depósitos reembolsables una vez que la subasta ha terminado. Emite `DepositWithdrawn`.
*   `withdrawExcess()`: Permite a los postores retirar el monto de su oferta que excede el mínimo requerido para mantener su posición, si su oferta ya no es la más alta. Emite `PartialRefund`.
*   `emergencyWithdraw()`: Permite al propietario retirar todos los fondos del contrato en una situación de emergencia, una vez que la subasta ha terminado. Emite `EmergencyWithdraw`.

### Funciones de Consulta (View Functions)

Estas funciones permiten a cualquier usuario consultar el estado actual de la subasta sin modificar su estado en la blockchain (funciones `view`).

```solidity
    function getHighestBid() external view returns (address bidder, uint amount);
    function getBidHistory() external view returns (Bid[] memory);
    function timeRemaining() external view returns (uint remainingTime);
    function bidsOf(address user) external view returns (uint[] memory userBids);
    function getDeadline() external view returns (uint auctionDeadline);
    function getOwner() external view returns (address contractOwner);
    function isEnded() external view returns (bool endedStatus);
    function totalBidOf(address user) external view returns (uint totalBidAmount);
```

*   `getHighestBid()`: Devuelve la dirección del postor actual con la oferta más alta y el monto de esa oferta.
*   `getBidHistory()`: Retorna el historial completo de todas las ofertas válidas registradas en la subasta.
*   `timeRemaining()`: Devuelve el tiempo restante en segundos hasta que la subasta finalice (0 si ya ha terminado).
*   `bidsOf(address user)`: Retorna un array con los montos de todas las ofertas realizadas por un usuario específico.
*   `getDeadline()`: Devuelve la marca de tiempo de la fecha límite de la subasta.
*   `getOwner()`: Devuelve la dirección del propietario del contrato.
*   `isEnded()`: Devuelve `true` si la subasta ha finalizado y `false` en caso contrario.
*   `totalBidOf(address user)`: Devuelve el monto total acumulado de todas las ofertas realizadas por un usuario específico.

## Consideraciones de Seguridad

El contrato ha sido diseñado teniendo en cuenta las siguientes consideraciones de seguridad:

*   **Patrón Checks-Effects-Interactions:** Las transferencias de Ether se realizan después de que se actualizan los estados internos del contrato para prevenir ataques de reentrada.
*   **Control de Acceso:** El uso del modificador `onlyOwner` asegura que solo el propietario pueda ejecutar funciones críticas.
*   **Validaciones Robustas:** Se utilizan `require` statements extensivamente para validar las entradas de las funciones y las condiciones de estado, previniendo así comportamientos inesperados y errores.
*   **Manejo de Errores:** Mensajes de error claros y específicos se proporcionan para guiar a los usuarios sobre las condiciones que no se cumplen.
*   **Dependencia de `block.timestamp`:** Aunque se utiliza `block.timestamp` para la lógica de tiempo, se reconoce que los mineros pueden manipular ligeramente este valor. Para subastas de alta seguridad, se podrían considerar oráculos de tiempo descentralizados.

## Despliegue y Uso

Para desplegar y utilizar este contrato, necesitarás un entorno de desarrollo de Solidity (como Truffle, Hardhat o Remix) y una red Ethereum (o una red de prueba como Sepolia, Goerli, etc.).

1.  **Compilación:** Compila el contrato `Auction.sol` utilizando el compilador de Solidity (versión `0.8.20` o superior).
2.  **Despliegue:** Despliega el contrato en la red de tu elección, proporcionando el parámetro del constructor (`_durationSeconds`).
3.  **Interacción:** Una vez desplegado, puedes interactuar con el contrato llamando a sus funciones a través de una interfaz de usuario (DApp) o directamente desde una consola de desarrollo.

**Ejemplo de Interacción (Pseudocódigo):**

```javascript
// Suponiendo que \'auction\' es una instancia del contrato desplegado

// Realizar una oferta
auction.placeBid({ value: web3.utils.toWei(\'0.5\', \'ether\') });

// Obtener la mejor oferta
const [bidder, amount] = await auction.getHighestBid();
console.log(`Mejor ofertante: ${bidder}, Mejor oferta: ${web3.utils.fromWei(amount, \'ether\')} ETH`);

// Finalizar la subasta (solo propietario)
auction.endAuction();

// Retirar fondos en exceso
auction.withdrawExcess();

// Reclamar reembolso (si no es el ganador)
auction.claimRefund();
```

## Licencia

Este proyecto está licenciado bajo la licencia MIT. Consulta el archivo `LICENSE` para más detalles.


