# État de l'art : Compression des images SVS en anatomopathologie numérique

## 1. Le format SVS

### 1.1 Définition et origine

Le format **SVS** (*ScanScope Virtual Slide*) est le format propriétaire historique des scanners **Aperio / Leica**. En pratique, un fichier `.svs` est un conteneur **TIFF pyramidal** spécialisé pour les images de lames entières (*Whole Slide Images*, WSI). Ces fichiers mesurent couramment **plusieurs gigaoctets** (ex. ~1,6 Go sur le jeu de données TCGA utilisé dans ce stage).

### 1.2 Structure interne

Un SVS n'est pas une image géante linéaire. Il est structuré en plusieurs niveaux :

- **Pyramide multi-résolution** : le niveau 0 est la pleine résolution (ex. 40x), suivi de niveaux sous-échantillonnés (4x, 16x, 64x plus petits) pour permettre un affichage fluide.
- **Tuilage** : chaque niveau est découpé en tuiles carrées (souvent 256×256 ou 512×512 pixels). Seules les tuiles visibles sont chargées en mémoire.
- **Images associées** : vignette (*thumbnail*), vue macro, étiquette de la lame.
- **Métadonnées** : fabricant, grossissement, résolution physique (microns/pixel), etc.

### 1.3 Pourquoi les fichiers sont si volumineux ?

Une lame numérisée à 40x peut atteindre **80 000 × 90 000 pixels** (niveau 0). Même compressé en JPEG dans chaque tuile, l'ensemble reste énorme car :
- l'image est en **RGB 24 bits** ;
- la compression JPEG d'origine est conservative (qualité élevée) pour préserver le diagnostic ;
- une grande partie de la surface est du **fond blanc** sans information utile, mais celui-ci reste encodé.

### 1.4 Conséquence pour la compression

Compresser un SVS avec ZIP ou GZIP donne peu de gain car les tuiles sont **déjà compressées**. Pour réduire réellement la taille, il faut donc **recompresser les tuiles** avec des paramètres plus agressifs, changer de codec, ou supprimer les zones inutiles.

---

## 2. Méthodes de compression

### 2.1 JPEG (baseline)

Le JPEG (*Joint Photographic Experts Group*, ISO/IEC 10918) est le codec le plus répandu. Il repose sur la DCT (transformée en cosinus discrète) et une quantification adaptée.

- **Avantage** : universel, rapide, compatible avec tous les visualiseurs.
- **Inconvénient** : artefacts de bloc visibles à fort taux ; perte irréversible.
- **Article de référence** : Chen et al., *Quantitative Assessment of the Effects of Compression on Deep Learning in Digital Pathology Image Analysis*, JCO Clinical Cancer Informatics, 2020. [Lien](https://pmc.ncbi.nlm.nih.gov/articles/PMC7113072/)

### 2.2 JPEG 2000

JPEG 2000 (ISO/IEC 15444) repose sur la **transformée en ondelettes** au lieu de la DCT. Il offre une scalabilité naturelle en résolution et qualité, et supporte le sans perte (5/3) et le avec perte (9/7).

- **Avantage** : meilleur ratio qualité/taille que JPEG à fort taux ; pas d'artefacts de bloc ; support du sans perte dans le même flux.
- **Inconvénient** : encodage/décodage plus lent ; compatibilité moindre avec les visualiseurs SVS classiques.
- **Article de référence** : Ghazvinian Zanjani et al., *Impact of JPEG 2000 compression on deep convolutional neural networks for metastatic cancer detection*, Journal of Medical Imaging, 2019. [Lien](https://pmc.ncbi.nlm.nih.gov/articles/PMC6479230/)

### 2.3 JPEG XL

JPEG XL (ISO/IEC 18181) est le codec le plus récent du consortium JPEG. Il promet des gains significatifs par rapport à JPEG et JPEG 2000, avec un support sans perte et une progressiveité améliorée.

- **Avantage** : très bon ratio qualité/taille ; décodage rapide ; support du sans perte.
- **Inconvénient** : adoption clinique quasi nulle à ce jour ; peu de visualiseurs médicaux le supportent.
- **Article de référence** : Alakuijala et al., *JPEG XL: Image Coding for the Future*, 2021 (disponible sur arXiv et sites du JPEG committee). [Lien](https://arxiv.org/abs/1908.03565)

### 2.4 WebP

WebP (Google) combine compression prédictive et transformée en cosinus. Il propose des modes avec et sans perte.

- **Avantage** : meilleur ratio que JPEG dans de nombreux cas ; open source.
- **Inconvénient** : rarement supporté dans les PACS et visualiseurs anatomopathologiques ; artefacts différents de JPEG.
- **Article de référence** : Google, *WebP Compression Study*, documentation technique. [Lien](https://developers.google.com/speed/webp/docs/compression)

### 2.5 AVIF

AVIF repose sur le codec vidéo AV1 (Alliance for Open Media). Il offre des taux de compression très agressifs.

- **Avantage** : parmi les meilleurs ratios qualité/taille actuels.
- **Inconvénient** : temps d'encodage très long ; support logiciel encore limité en pathologie numérique.
- **Article de référence** : Alliance for Open Media, *AV1 Image File Format (AVIF)*, 2019. [Lien](https://aomediacodec.github.io/av1-avif/)

### 2.6 Compression sans perte (PNG, TIFF Deflate, JPEG-LS)

Ces méthodes garantissent une restitution exacte des pixels.

- **Avantage** : aucune perte d'information ; obligatoire pour certains contextes légaux ou de recherche clinique stricte.
- **Inconvénient** : gain limité sur des images déjà compressées en JPEG ; ne résout pas le problème des téraoctets de stockage.
- **Article de référence** : Helin et al., *Optimized JPEG 2000 Compression for Efficient Storage of Histopathological Whole-Slide Images*, 2018. [Lien](https://pmc.ncbi.nlm.nih.gov/articles/PMC5989536/)

### 2.7 Compression adaptative et suppression du fond

Plutôt qu'uniforme, la compression peut être adaptée au contenu : zones de tissu = haute qualité, fond blanc = forte compression ou suppression. Codipilly et al. proposent de détecter le tissu, de l'extraire et de le reconditionner dans un fichier plus compact.

- **Avantage** : réduction très forte (jusqu'à ×7 rapportée) quand la lame contient beaucoup de blanc.
- **Inconvénient** : la géométrie originale est modifiée ; moins adapté à l'archivage clinique strict.
- **Article de référence** : Codipilly et al., *Optimizing Storage and Computational Efficiency: An Efficient Algorithm for Whole Slide Image Size Reduction*, Mayo Clinic Proceedings: Digital Health, 2023. [Lien](https://pmc.ncbi.nlm.nih.gov/articles/PMC11975739/)

---

## 3. Tableau comparatif

| Méthode | Perte ? | Ratio typique | Vitesse | Usage médical | Référence principale |
|---|---|---|---|---|---|
| JPEG | Oui | Moyen | Très rapide | Universel | Chen et al., 2020 |
| JPEG 2000 | Oui / Non | Bon | Lent | Standard Aperio/Philips | Zanjani et al., 2019 |
| JPEG XL | Oui / Non | Très bon | Rapide | Faible adoption | Alakuijala et al., 2021 |
| WebP | Oui / Non | Bon | Rapide | Faible | Google WebP Study |
| AVIF | Oui | Très bon | Lent (enc.) | Quasi nulle | AOM AVIF Spec |
| Sans perte | Non | Faible | Variable | Archives légales | Helin et al., 2018 |
| Adaptative (fond) | Variable | Très bon | Modéré | Recherche / IA | Codipilly et al., 2023 |

---

## 4. Objectif du benchmark

Ce stage se concentre sur une approche pragmatique :
1. **Caractériser** un échantillon de lames SVS (taille, structure, résolution).
2. **Tester** plusieurs codecs et niveaux de qualité sur des tuiles représentatives.
3. **Mesurer** le compromis taille / qualité / temps.
4. **Estimer** l'impact écologique (To économisés, kWh et kgCO₂e évités).

Les scripts associés à ce rapport (`01_analyse_svs.py` à `05_visualisation.py`) implémentent ces étapes de manière reproductible.
