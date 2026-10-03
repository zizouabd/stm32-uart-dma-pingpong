# Communication UART Optimisée DMA & Protocole Ping-Pong (STM32 Black Pill)

Ce projet implémente un pilote de communication série asynchrone hautement efficace sur **STM32 Black Pill (STM32F411)**. Il utilise le contrôleur **DMA** couplé à la détection de ligne inactive (**Receive-to-Idle**) pour traiter un protocole de trame binaire avec un coût processeur quasi nul.

---

## 📋 Spécifications Générales

* **Microcontrôleur :** STM32 Black Pill
* **Peripherique :** USART1 
* **Paramètres UART :** 115200 Bauds, 8 bits de données, Sans parité, 1 bit de Stop (115200 8N1)
* **Target CPU Usage :** Minimisé au maximum via DMA hardware

---

## 📐 Spécification du Protocole de Trame

Chaque paquet transmis ou reçu respecte la structure binaire suivante :

| Champ | Taille | Valeur | Description |
| :--- | :--- | :--- | :--- |
| **HEAD** | 1 octet | `0xAA` | Marqueur de début de trame |
| **CMD** | 1 octet | `0x00` - `0xFF` | Identifiant de commande |
| **LEN** | 1 octet | `0x00` - `0x20` | Taille du payload ($\le 32$ octets) |
| **PAYLOAD** | `LEN` octets | Variable | Données utiles de la commande |
| **CHK** | 1 octet | Variable | XOR Checksum (calculé sur CMD, LEN, PAYLOAD) |
| **STOP** | 1 octet | `0x00` | Marqueur de fin de trame |

La taille totale d'une trame est définie par l'équation :
$$T_{\text{taille}} = \text{LEN} + 5 \text{ octets}$$

Le paquet complet ne dépasse jamais $37 \text{ octets}$ ($\text{LEN}_{\max} = 32$).

### Formule du Checksum
Le checksum 8 bits est obtenu en effectuant un OU exclusif ($\oplus$) successif :
$$CHK = CMD \oplus LEN \oplus \bigoplus_{i=0}^{LEN-1} PAYLOAD[i]$$

---

## 🧠 Décisions Techniques & Architecture Logicielle

### 1. Minimisation du temps CPU (Receive-to-Idle DMA)
* **Problème :** Les interruptions par octet génèrent un overhead important à 115200 bauds. Le Polling (scrutation) bloque inutilement l'exécution du microcontrôleur.
* **Solution retenue :** Utilisation de `HAL_UARTEx_ReceiveToIdle_DMA`. Le contrôleur DMA remplit le buffer en arrière-plan sans solliciter le CPU. Le CPU n'est réveillé via une interruption que lorsque la ligne UART redevient inactive (pause inter-trame), signalant la fin de la réception.

### 2. Validation Sécurisée des Données
À la réception d'une trame, la fonction `Process_Incoming_Data` applique les filtres suivants :
1. **Recherche de l'en-tête :** Localisation de l'octet `0xAA` (`HEAD`).
2. **Contrôle de taille :** Vérification stricte que $\text{LEN} \le 0\times20$ pour éviter tout débordement de mémoire de buffer.
3. **Contrôle de fin :** Validation de l'octet de fin `0x00` (`STOP`).
4. **Intégrité :** Comparaison du checksum calculé avec l'octet `CHK` reçu. Si une erreur est détectée, le paquet est immédiatement rejeté.

### 3. Logique Ping-Pong
Lorsqu'un paquet valide est réceptionné :
* Le système extrait les données.
* Il applique un masque binaire sur la commande (`CMD | 0x80`) pour signifier une réponse (Pong).
* Il réémet la trame de manière asynchrone via DMA (`HAL_UART_Transmit_DMA`).
* La réception DMA est relancée instantanément pour intercepter la trame suivante.

---
