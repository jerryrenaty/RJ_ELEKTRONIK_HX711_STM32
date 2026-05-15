# Guide d'Utilisation du Driver HX711 pour STM32 HAL

Ce module permet de gérer le convertisseur analogique-numérique 24 bits **HX711** dédié aux balances et aux capteurs de pesée industriels.

---

## 📌 1. Configuration Matérielle (CubeMX)

Avant d'utiliser le code, configurez vos broches dans STM32CubeMX :

*   **Broche SCK (Horloge) :** 
    *   Mode : `GPIO_Output`
    *   Vitesse : `High Speed` ou `Very High Speed`
    *   Niveau initial : `Low`
*   **Broche DOUT / DT (Données) :**
    *   Mode : `GPIO_Input`
    *   Pull-up/Pull-down : `No pull-up and no pull-down`

---

## 🛠️ 2. Étape Essentielle : La Calibration

Le capteur renvoie une valeur numérique brute. Pour obtenir un poids en grammes, vous devez calculer le `calibration_factor`.

### Protocole de calcul du facteur :
1. Démarrez la balance à vide et lancez la fonction `HX711_Tare()`.
2. Déposez un objet dont vous connaissez le poids exact (ex: un poids de 500g).
3. Lisez la valeur brute renvoyée par le capteur via `HX711_Read()`.
4. Utilisez la formule suivante :
   $$\text{Facteur de calibration} = \frac{\text{Valeur Brute} - \text{Offset de la Tare}}{\text{Poids Réel en Grammes}}$$
5. Configurez cette valeur dans la structure : `hx711.calibration_factor = calcul_facteur;`

---

## 💻 3. Exemple Complet d'Intégration (`main.c`)

Voici comment intégrer le driver dans votre boucle principale STM32 :

```c
#include "main.h"
#include "RJ_ELEKTRONIK_HX711_STM32.h"
#include <stdio.h>

// Instance globale du capteur
HX711_HandleTypeDef MyScale;
float current_weight = 0.0f;
char uart_buffer[50];

int main(void) {
    // Initialisations génériques du système STM32 HAL
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART1_UART_Init();

    // 1. Initialisation du module (Port A, Pin 0 pour CLK / Pin 1 pour DT)
    HX711_Init(&MyScale, GPIOA, GPIO_PIN_0, GPIOA, GPIO_PIN_1, HX711_GAIN_128);

    // 2. Configuration du facteur calculé pendant la phase de calibration
    MyScale.calibration_factor = 423.5f; // Exemple de valeur

    // 3. Réalisation de la tare au démarrage (Moyenne sur 10 mesures)
    HAL_UART_Transmit(&huart1, (uint8_t*)"Tare en cours...\r\n", 18, 100);
    HX711_Tare(&MyScale, 10);
    HAL_UART_Transmit(&huart1, (uint8_t*)"Balance Prete !\r\n", 17, 100);

    while (1) {
        // 4. Lecture du poids lissé sur 5 échantillons
        current_weight = HX711_GetWeightGram(&MyScale, 5);

        // 5. Affichage du résultat sur le port série UART
        sprintf(uart_buffer, "Poids : %.2f g\r\n", current_weight);
        HAL_UART_Transmit(&huart1, (uint8_t*)uart_buffer, strlen(uart_buffer), 100);

        HAL_Delay(500); // Rafraîchissement toutes les 500ms
    }
}
```

---

## 🔍 4. Résolution des Problèmes Récurrents

*   **Le poids reste bloqué à 0 :** Vérifiez le câblage électrique du pont de jauge (E+, E-, A+, A-). Un faux contact coupe la transmission.
*   **La mesure fluctue énormément :** Augmentez le nombre d'échantillons (paramètre `samples` passé à `HX711_GetWeightGram`). Assurez-vous que l'alimentation 5V/3.3V du module est stable et filtrée.
*   **Les valeurs augmentent quand on retire du poids :** Inversez les fils de signal `A+` et `A-` de votre jauge de contrainte sur le module HX711.
