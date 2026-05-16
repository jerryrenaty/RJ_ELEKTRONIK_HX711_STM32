# Driver HX711 pour STM32 (HAL)

Ce dépôt contient un driver optimisé et corrigé pour le convertisseur analogique-numérique 24 bits **HX711**, spécifiquement conçu pour les microcontrôleurs STM32 utilisant la bibliothèque **HAL**. 

Il intègre des correctifs majeurs concernant l'extension de signe (24 bits vers 32 bits), la gestion des gains, et la sécurité temporelle pour éviter la mise en veille accidentelle du composant.

## ✨ Fonctionnalités
* ⚙️ **Configuration Simplifiée** : Utilisation directe des macros matérielles définies dans CubeMX.
* 🛡️ **Lecture Sécurisée** : Désactivation temporaire des interruptions pendant la communication critique pour éviter le mode *Power Down* du HX711.
* 🔄 **Correction de Bit** : Alignement de décalage de bits corrigé pour garantir la précision du poids faible.
* ⚖️ **Zéro Flexible** : Prise en charge des valeurs négatives de pesée (suppression du blocage à `0.0f`).

## 📌 Brochage par Défaut
Le driver est configuré par défaut sur le port **GPIOB**. Si vous utilisez d'autres broches, modifiez-les dans le fichier `main.h` (via CubeMX) ou directement dans le fichier `RJ_ELEKTRONIK_HX711_STM32.h`.


| Signal HX711 | Broche STM32 | Fonction |
| :--- | :--- | :--- |
| **SCK** (Clock) | `GPIOB - Pin 12` | Sortie Horloge numérique |
| **DT** (Data) | `GPIOB - Pin 13` | Entrée Données numériques |

## 🚀 Guide d'Utilisation rapide

### 1. Structure globale et Initialisation
Déclarez le handle de gestion du HX711 dans votre fichier `main.c`, puis initialisez-le avec le gain souhaité (ex: `HX711_GAIN_128`).

```c
#include "RJ_ELEKTRONIK_HX711_STM32.h"

HX711_HandleTypeDef hx711;

int main(void) {
    // Initialisation du matériel STM32 (HAL_Init, SystemClock_Config, MX_GPIO_Init...)
    HAL_Init();
    
    // Initialisation du driver HX711
    HX711_Init(&hx711, HX711_GAIN_128);
    
    // Effectuer une tare automatique (20 échantillons)
    HX711_Tare(&hx711, 20);

    while (1) {
        // Lecture du poids toutes les 200ms (moyenne sur 5 échantillons)
        float poids = HX711_GetWeightGram(&hx711, 5);
        
        // Votre code d'affichage (ex: UART, LCD ou OLED)
        
        HAL_Delay(200);
    }
}
```

### 2. Calibration de la balance
Pour afficher un poids exact en grammes, vous devez configurer la variable `calibration_factor`. 

1. Laissez la balance à vide et démarrez le programme (la fonction `HX711_Tare` fixe le point zéro).
2. Placez un objet dont vous connaissez précisément le poids (ex: un poids étalon de 500g).
3. Lisez la valeur brute renvoyée par `hx711.last_raw_value`.
4. Calculez le facteur : `calibration_factor = (Valeur Brute - Valeur Offset) / Poids Réel`.
5. Appliquez ce facteur juste après l'initialisation dans votre code :
   ```c
   hx711.calibration_factor = 423.5f; // Remplacez par votre valeur calculée
   ```

## 🛠️ Fonctions disponibles

* `void HX711_Init(HX711_HandleTypeDef *hx711, HX711_Gain initial_gain)`  
  Initialise la structure, configure les GPIOs (SCK en sortie, DT en entrée) et applique le gain initial.
* `int32_t HX711_Read(HX711_HandleTypeDef *hx711)`  
  Lit une valeur brute 24 bits signée directement depuis le capteur.
* `void HX711_Tare(HX711_HandleTypeDef *hx711, uint8_t samples)`  
  Calcule la moyenne des échantillons à vide pour définir l'offset (Zéro).
* `float HX711_GetWeightGram(HX711_HandleTypeDef *hx711, uint8_t samples)`  
  Convertit la lecture brute filtrée en grammes à l'aide du facteur de calibration.
* `void HX711_SetGain(HX711_HandleTypeDef *hx711, HX711_Gain new_gain)`  
  Modifie logiciellement et matériellement le gain/canal de lecture du HX711.

## 📝 Licence
Ce projet est distribué sous licence MIT. Libre à vous de l'utiliser et de l'adapter dans vos projets commerciaux ou personnels.
