# Optiver Realized Volatility Prediction — carnet de recherche

État au 17 novembre 2026. Cette note conserve les idées et résultats obtenus pendant notre exploration. Les chiffres ci-dessous sont des scores **locaux** ; ils ne sont pas un score de soumission sur les données cachées. Les expériences retenues portent toujours sur les **112 actions**.

## Question et protocole

À partir de 600 secondes de carnets et de transactions par `(time_id, stock_id)`, prédire la volatilité réalisée de la période future. La métrique utilisée est la RMSPE : `sqrt(mean((prediction / target - 1)**2))` sur les lignes à cible positive.

Notre validation a cinq plis par `time_id` sur 3 064 fenêtres (343 145 lignes), puis un holdout de 766 fenêtres (85 787 lignes), soit 3 830 fenêtres. Les actions d'une même fenêtre restent ensemble. Les poids du mélange sont choisis sans consulter la cible holdout. Cette validation aléatoire par fenêtre ne garantit pas le même résultat sur la période privée future.

## Résultats conservés

| Modèle, 112 actions | OOF RMSPE | Holdout RMSPE |
| --- | ---: | ---: |
| Ancien GRU avec marché entier | 0,208702 | 0,212209 |
| LightGBM avec voisins de prix, RV et transactions, mélange 25 % avec ancien marché (400 arbres) | 0,205224 | 0,209787 |
| Même mélange, 800 arbres | 0,205058 | 0,209587 |
| LightGBM seul avec nouvelles features inspirées du premier, 800 arbres | 0,196252 | 0,202839 |
| GRU réentraîné avec ces features | 0,193569 | 0,196240 |
| **GRU + LightGBM, poids LightGBM = 0,38753884496731694** | **0,191787** | **0,195591** |
| GNN réentraîné avec les mêmes features | 0,197824 | 0,203218 |

Le mélange fixe utilise `0.6124611550326831 * p_gru + 0.38753884496731694 * p_lgb`. Sa recomposition à partir des prédictions sauvegardées a été contrôlée : écart maximal 0,0 ; holdout recalculé sur les 112 actions, 0,19559124763851463.

### Ce qui a débloqué le modèle

Nous avions beaucoup travaillé la séquence de 10 secondes, les messages du graphe et des corrections de volatilité commune. Le saut de qualité est venu de **l'ingénierie de caractéristiques entre actions et entre fenêtres**, étudiée à partir de la solution gagnante de nyanp (liens ci-dessous), puis donnée à **deux modèles réentraînés**.

Les composantes importantes de cette famille :

- Statistiques du carnet et des transactions sur la fenêtre complète et sur les segments commençant à 150, 300 et 450 secondes : volatilité des log-rendements de WAP et des quotes, spread, profondeur, volume, activité des ordres.
- Rangs au sein de chaque `time_id` pour des volumes et fréquences d'activité : décrire la position relative d'une action dans son marché.
- Prix reconstitué, puis voisins de fenêtres par prix, volatilité et taille/activité ; voisins d'actions également. Agrégation de la volatilité et de l'activité observées chez ces voisins à plusieurs tailles de voisinage.
- Rapports entre mesure propre et mesures des voisins, notamment `vol_rankmin` et `vol_rankmax`. Quelques moyennes glissantes par action et un petit embedding d'identité ont aussi été essayés.

L'ablation LightGBM, à plis et lignes identiques, est passée de **0,208325 à 0,192664** pour le modèle seul sur les lignes couvertes historiquement ; en mélange à 25 % avec l'ancien réseau, de **0,205058 à 0,201431** sur les 112 actions. Le LightGBM seul avec couverture complète fait 0,196252 OOF. Le GRU réentraîné avec les nouvelles caractéristiques, y compris les 112 actions, atteint 0,193569 OOF et 0,196240 holdout. Le modèle séquentiel bénéficie donc nettement d'une représentation tabulaire des états comparables.

Une attribution exploratoire du GRU place la famille voisins/rangs avant carnet, historique et transactions ; elle ne mesure pas à elle seule un effet causal. En réentraînant le GRU sans voisins/rangs sur deux plis, la RMSPE s'est dégradée de 0,011291 et 0,005457. Les sept canaux séquentiels les moins attribués, retirés ensemble, changent peu l'OOF mais dégradent le holdout de 0,001083 : on les conserve. Ces chiffres concernent des ablations limitées, pas une sélection exhaustive de features.

## Autres pistes étudiées

- **Ordre des fenêtres.** Les 3 830 `time_id` ne démontrent pas une cadence de 30 minutes ; les créneaux nécessaires par jour dépassent la capacité de certaines hypothèses. La continuité de prix entre fenêtres sélectionne des marchés semblables et favorise parfois le sens fin → début, mais les chaînes, cycles et contrôles sur actions indépendantes n'établissent pas une chronologie fiable. La piste des paires a été testée puis laissée de côté : le gain de correction résiduelle était fragile et concentré sur peu de fenêtres.
- **Voisins entre fenêtres.** Le signal de volatilité observée des voisins de prix subsiste après le premier modèle. L'ajout de voisinages par prix, RV et transactions a régulièrement amélioré les cinq plis et le holdout. Les voisins de transactions apportaient davantage que les seuls voisins de RV dans une ablation comparable. Il faut bien distinguer les features de carnets observés, disponibles pour le lot d'inférence, des *targets futures*, réservées aux diagnostics rétrospectifs.
- **Mouvement collectif futur.** Un facteur commun entre actions est réel mais difficile à prévoir à partir du seul passé observé. Après le GRU et le mélange, les grandes hausses collectives manquées restent coûteuses : 412 fenêtres OOF et 126 holdout dans un audit GRU, représentant respectivement 18,89 % et 24,06 % de la perte totale. Les erreurs GRU et LightGBM restent fortement liées (corrélation des résidus par ligne d'environ 0,928). Une correction commune par 38 résumés dégrade l'OOF du mélange de 0,191787 à 0,192170, malgré un holdout légèrement meilleur : elle n'est pas retenue.
- **Queue et action 31.** Quelques chutes de cible très fortes sur une action causent une perte disproportionnée ; l'action 31 se démarque. Elle doit néanmoins rester dans l'évaluation et dans toute soumission, comme les 111 autres. Les calibrations de queue et les seuils de risque n'ont pas donné de correctif assez robuste.
- **Autres essais.** GNN avec les mêmes features inférieur au GRU ; correction par dispersion des voisins sans pouvoir de classification utile ; poids de mélange adaptatifs selon transactions/carnet presque identiques au mélange fixe ; tâche auxiliaire 300 secondes donnant un apport propre d'environ +0,000021 OOF et −0,000023 holdout, donc sans gain stable.

## Reproductibilité : état réel

Les prédictions OOF/holdout et le poids de mélange sont sauvegardés côté Kaggle sous `/kaggle/working/optiver_reprise/gru_lgb_top1_complementarite/comparaison.npz` (clés `p_gru`, `p_lgb`, `p_melange`, `y`, `plis`, `time_ids`, `stock_ids`, `poids_lgb_final`). La recette figée est sous `blend_gru_lgb_top1_final`. Les checkpoints du GRU avec nouvelles features existent dans `gru_avec_fe_top1/final_holdout.pt` et `pli_1.pt` à `pli_5.pt` ; les features sous `reproduction_top1_fe/features_finales.parquet` ; la séquence sous `sequence_micro_10s_112.npy`.

**Les tableaux NPZ de validation ne fournissent aucune prédiction pour de nouveaux `time_id` cachés.** Pour soumettre exactement le mélange 0,195591, il reste à sauvegarder le code d'architecture, de prétraitement et d'inférence du GRU ainsi que le modèle final LightGBM correspondant aux nouvelles features ; puis à reproduire, avec ce *même* pipeline, le GRU 0,196240, le LightGBM 0,202839 et le mélange 0,195591 sur holdout, ligne par ligne et dans l'ordre `row_id`. L'existence des poids GRU a été vérifiée ; l'affirmation antérieure selon laquelle ils avaient disparu était fausse.

Une ancienne soumission a reçu **0,24849**, mais elle utilisait une **référence distillée différente**, dont le holdout valait 0,242627 au lieu de la référence originale 0,212223. Son mauvais score ne valide ni n'invalide le nouveau mélange. Aucun score public ou privé du mélange 0,195591 n'a été observé ici.

## Crédit et suite

Cette recherche s'appuie fortement sur les idées et le code publiés par **nyanp** : [solution gagnante, « 1st Place Solution — Nearest Neighbors »](https://www.kaggle.com/competitions/optiver-realized-volatility-prediction/discussion/274970) et [notebook de solution](https://www.kaggle.com/code/nyanpn/1st-place-public-2nd-place-solution). Les résultats, réentraînements GRU/GNN, comparaisons OOF/holdout et erreurs de reproductibilité décrits ici concernent notre propre expérience. Si du code de la solution est ajouté au dépôt, conserver aussi son attribution et sa licence éventuelle.

Idée à réutiliser : quand une représentation temporelle atteint un plafond, tester si la bonne unité de comparaison est le **voisin observable dans l'espace des états du marché**, puis le donner au modèle séquentiel. La preuve doit rester une ablation sur les mêmes lignes, suivie d'un holdout et d'une inférence reproductible.
