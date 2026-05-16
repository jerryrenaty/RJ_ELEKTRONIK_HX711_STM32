# ⚖️ Driver HX711 Stabilisé et Calibré pour STM32 (HAL)

Ce dépôt contient une implémentation optimisée, sécurisée et entièrement corrigée du driver pour le convertisseur analogique-numérique 24 bits **HX711**, destinée aux microcontrôleurs STM32 utilisant la bibliothèque **HAL**.

Ce projet intègre des correctifs critiques pour éliminer les dérives de valeurs et synchronise les lectures en temps réel avec un afficheur **7 segments** et un terminal de **débogage série USB**.

## ✨ Fonctionnalités Majeures
* 🛡️ **Immunité Électrique Augmentée** : Configuration de la broche de données (`DT`) en mode `GPIO_PULLUP` pour éliminer le bruit électromagnétique ambiant et les parasites générés par le multiplexage de l'afficheur 7 segments.
* 🕒 **Communication Sécurisée (Bit-Banging)** : Intégration d'un micro-délai matériel de 5 µs (`HX711_DELAY`) pour adapter le rythme du STM32 (très rapide) aux spécifications temporelles du HX711.
* 🔒 **Section Critique Protégée** : Désactivation temporaire des interruptions (`__disable_irq()`) pendant la capture des 24 bits pour empêcher le HX711 d'entrer en mode *Power Down* accidentel.
* 🌊 **Filtrage par Moyenne Glissante** : Algorithme logiciel de lissage (fenêtre de 5 échantillons) pour figer l'affichage et supprimer les micro-oscillations du dernier chiffre.
* 🔄 **Correction de Signe & Alignement** : Alignement de décalage de bit corrigé et extension de signe (24 bits vers 32 bits complément à 2) fonctionnelle, autorisant les valeurs négatives de pesée.
* 🔗 **Synchronisation Totale** : Mise à jour simultanée (toutes les 50 ms) de l'afficheur matériel 7 segments et du port série.

## 📌 Schéma de Câblage par Défaut

Les broches matérielles sont fixées globalement (générées par STM32CubeMX ou définies statiquement dans le header).


| Signal HX711 | Broche STM32 | Configuration GPIO | Fonction |
| :--- | :--- | :--- | :--- |
| **SCK** (Clock) | `GPIOB - Pin 12` | Output Push-Pull / High Speed | Horloge numérique |
| **DT** (Data) | `GPIOB - Pin 13` | Input / Pull-Up interne | Données numériques |

> ⚠️ **Note Électrique** : Le multiplexage d'un afficheur 7 segments consomme beaucoup de courant et peut polluer l'alimentation du HX711. Si des instabilités persistent, connectez le HX711 sur le 5V et l'afficheur sur le 3.3V de votre STM32, et ajoutez un condensateur de découplage de `10 µF` au plus près du module HX711.

## 🚀 Guide d'Intégration Rapide (`main.c`)

### 1. Variables globales et Filtre (`USER CODE BEGIN 0`)
```c
#include "RJ_ELEKTRONIK_HX711_STM32.h"

char tx_buffer[64]; 
float poids_brut;
float poids_filtre;

#define FILTRE_TAILLE 5
float historique_poids[FILTRE_TAILLE] = {0.0f};
uint8_t index_filtre = 0;

float Filtrer_Poids(float nouvelle_valeur) {
    historique_poids[index_filtre] = nouvelle_valeur;
    index_filtre = (index_filtre + 1) % FILTRE_TAILLE;
    
    float somme = 0.0f;
    for (uint8_t i = 0; i < FILTRE_TAILLE; i++) {
        somme += historique_poids[i];
    }
    return somme / FILTRE_TAILLE;
}
```

### 2. Initialisation et Calibration (`USER CODE BEGIN 2`)
Le facteur ci-dessous (`91.07f`) a été calculé et validé à l'aide d'un smartphone *Xiaomi Redmi A3* (poids officiel de 199 g).
```c
HAL_Delay(3000); // Laisse l'alimentation globale se stabiliser
Serial_debug_RJ();

// Initialisation de l'affichage 7 segments piloté par Timer (Interruption)
SevenSeg_Init(COMMON_ANODE, &htim3);
HAL_TIM_Base_Start_IT(&htim3);

// Initialisation du capteur de poids
HX711_Init(&hx711, HX711_GAIN_128);
hx711.calibration_factor = 91.07f; // Facteur d'étalonnage précis

HAL_Delay(500);
HX711_Tare(&hx711, 20); // Fixe le point zéro à vide sur 20 échantillons
```

### 3. Boucle Principale Synchrone (`USER CODE BEGIN 3`)
```c
while (1)
{
    // 1. Lecture native du HX711 (Moyenne matérielle rapide sur 3 échantillons)
    poids_brut = HX711_GetWeightGram(&hx711, 3);

    // 2. Lissage logiciel par moyenne glissante
    poids_filtre = Filtrer_Poids(poids_brut);

    // 3. Envoi simultané sur l'afficheur matériel
    SevenSeg_UpdateNumber(poids_filtre, false, 2);

    // 4. Arrondi mathématique pour le formatage textuel entier
    int32_t poids_entier = (int32_t)(poids_filtre + 0.5f);
    if (poids_filtre < 0) {
        poids_entier = (int32_t)(poids_filtre - 0.5f);
    }

    // 5. Envoi simultané sur le port série USB (Synchronisé à l'afficheur)
    int len = snprintf(tx_buffer, sizeof(tx_buffer), "Poids : %ld g\r\n", poids_entier);
    if (len > 0) {
       debug_print(tx_buffer);
    }

    HAL_Delay(50); // Cadence d'affichage fluide de 20 Hz
}
```

## 🛠️ API du Driver

* `void HX711_Init(HX711_HandleTypeDef *hx711, HX711_Gain initial_gain)`  
  Configure les structures logicielles, règle le gain initial et initialise les GPIOs (SCK en sortie, DT en entrée Pull-Up).
* `int32_t HX711_Read(HX711_HandleTypeDef *hx711)`  
  Gère l'attente active du composant, coupe les interruptions et extrait la valeur 24 bits signée via une communication synchrone ralentie.
* `void HX711_Tare(HX711_HandleTypeDef *hx711, uint8_t samples)`  
  Effectue une suite de lectures au rythme natif du circuit pour définir la valeur de décalage électrique (`offset`) à vide.
* `float HX711_GetWeightGram(HX711_HandleTypeDef *hx711, uint8_t samples)`  
  Renvoie la masse convertie en grammes en appliquant l'offset et le facteur de calibration.
* `void HX711_SetGain(HX711_HandleTypeDef *hx711, HX711_Gain new_gain)`  
  Modifie le gain matériel (Canal A 128/64 ou Canal B 32) pour le prochain cycle de conversion.

## 📝 Licence
Ce projet est open-source et distribué sous licence MIT. Vous pouvez librement l'utiliser, le modifier et le distribuer pour vos projets personnels ou industriels.
