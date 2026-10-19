# GLM — Study Notes

Source: [GLM.html](GLM.html)

## Data Preparation

### Niveau de référence

Pour une variable catégorielle, choisir comme niveau de référence celui qui a l’exposition totale la plus élevée, plutôt que le premier niveau, que R utilise par défaut. [Voir la section Data Preparation](GLM.html#data-preparation).

Exception : si la variable est ordonnée et que les données sont suffisantes, on peut choisir son premier niveau comme référence.

L’exemple de `Power` illustre cette exception : les niveaux sont ordonnés de la puissance la plus faible à la plus élevée, et `d`, le premier niveau, est conservé comme référence pour faciliter l’interprétation des coefficients. Bien que `f` soit le niveau le plus fréquent, `d` reste suffisamment représenté. [Voir l’exemple Power](GLM.html#power).

Pour rétablir l’ordre croissant des niveaux de `Power` et placer `d` en premier, on peut utiliser `fct_relevel(sort)`.

Pour réordonner automatiquement les niveaux selon leur fréquence, utiliser `fct_infreq()` du package `forcats` : le niveau le plus fréquent devient le premier, donc le niveau de référence. Cette fonction se base sur le nombre d’observations, et non sur l’exposition totale. [Voir l’exemple sur Brand](GLM.html#brand).

## Model

### Training et test sets

Créer aléatoirement un training set et un test set : les données ne sont pas temporelles et les observations sont supposées indépendantes. Le cours utilise `caret::createDataPartition()` pour répartir les données en 80 % pour l’entraînement et 20 % pour le test, avec `ClaimNb` comme variable de partitionnement. Fixer une graine avec `set.seed(21)` permet de reproduire le tirage. [Voir la section Model](GLM.html#model).

Attention : séparer les données avant d’ajuster les étapes de préparation qui apprennent des paramètres à partir des observations (p. ex. moyenne et écart-type pour standardiser, imputation ou regroupement de niveaux). Ajuster ces étapes sur le training set uniquement, puis les appliquer au test set sans les réestimer, afin d’éviter la fuite d’information. Les transformations déterministes définies indépendamment des données peuvent être faites en amont. [Voir la section Restart with recipes](GLM.html#restart-with-recipes).

Par exemple, pour centrer-réduire une variable, calculer sa moyenne et son écart-type sur le training set, puis réutiliser ces mêmes valeurs pour transformer le test set. Recalculer ces paramètres sur le test donnerait une échelle différente; les calculer sur l’ensemble complet ferait fuiter de l’information du test vers l’entraînement.

Pour rendre ce processus reproductible et compatible avec la validation croisée, regrouper les étapes dans une `recipe` : `prep()` apprend les transformations sur les données d’entraînement, puis `bake(rec_0, new_data = dataset)` applique la recette préparée au dataset indiqué. On peut ainsi transformer le training set et le test set avec la même recette. Dans chaque pli de validation croisée, la recette doit être réajustée sur le pli d’entraînement puis appliquée au pli de validation, afin d’éviter toute fuite d’information. [Voir l’exemple avec recipes](GLM.html#restart-with-recipes).

### Workflow avec tidymodels

Un workflow regroupe la préparation des données et la spécification du modèle, puis permet d’ajuster l’ensemble :

1. Définir la recette `rec_4` de préparation.
2. Définir `m4_mod = poisson_reg() %>% set_engine("glm")` avec `parsnip`.
3. Assembler les deux dans `m4_wflow` avec `workflow() %>% add_recipe(rec_4) %>% add_model(m4_mod, formula = ClaimNb ~ Power + Gas + Region + Brand + offset(log(Exposure)))`.
4. Appeler `m4_fit = fit(m4_wflow, data = training_set)` : le workflow applique la recette, puis ajuste le modèle.

[Voir la section Workflows et parsnip](GLM.html#use-workflows-and-parsnip-to-specify-model).

### Intercept

Rappel théorique : le nombre de sinistres de l’observation $i$ suit une loi de Poisson de paramètre $d_i \exp(\text{score}_i)$, où $d_i$ est l’exposition :

$$N_i \sim \operatorname{Poisson}\left(d_i \exp(\text{score}_i)\right), \qquad d_i = \text{exposition}_i.$$

Le score (prédicteur linéaire) s’écrit :

$$\text{score}_i = \beta_0 + \sum_j \beta_j x_{ij}.$$

Dans le premier modèle, qui ne contient qu’un intercept, $\text{score}_i = \beta_0$. Le paramètre de Poisson est donc $\lambda_i = d_i \exp(\beta_0)$, soit :

$$N_i \sim \operatorname{Poisson}\left(d_i \exp(\beta_0)\right), \qquad \log \lambda_i = \beta_0 + \log d_i.$$

Le terme $\log d_i$ est l’offset (log-exposition) du modèle : son coefficient est fixé à 1 et n’est donc pas estimé par `glm`. Ainsi, $\exp(\beta_0)$ représente la fréquence moyenne des sinistres.

Avec `family = poisson()`, `glm` utilise automatiquement le lien logarithmique (`log`), qui est le lien canonique de la loi de Poisson. On peut aussi le préciser explicitement avec `poisson(link = "log")`.

Avec ce lien canonique, l’équation de score de l’intercept implique que la somme des observations est égale à la somme des moyennes ajustées :

$$\sum_i y_i = \sum_i \widehat{\mu}_i, \qquad \widehat{\mu}_i = d_i \exp(\widehat{\beta}_0).$$

On en déduit, pour l’échantillon d’entraînement :

$$\exp(\widehat{\beta}_0) = \frac{\sum_i y_i}{\sum_i d_i} = \frac{\text{somme des sinistres}}{\text{somme des expositions}}.$$

Ainsi, $\exp(\widehat{\beta}_0)$ est la fréquence empirique globale des sinistres, calculée par unité d’exposition.

[Voir la section Intercept](GLM.html#intercept).

### Variables explicatives

En ajoutant des variables explicatives catégorielles, les coefficients associés à leurs niveaux quantifient l’effet de chaque catégorie par rapport à sa catégorie de référence, toutes choses égales par ailleurs. Avec le lien logarithmique, le coefficient $\beta$ est un log-rapport de fréquences; $\exp(\beta)$ donne le rapport multiplicatif des fréquences prédites entre la catégorie considérée et la référence. [Voir la section Include Variables](GLM.html#include-variables).

#### Visualisation avec `visreg`

`visreg` permet de tracer les effets sous forme conditionnelle (`type = "conditional"`) ou par contraste (`type = "contrast"`). En mode conditionnel, les autres variables sont maintenues constantes : les variables numériques sont fixées à leur médiane et les facteurs à leur modalité la plus fréquente. En mode contraste, l’effet est exprimé relativement à la prédiction au point de référence, plutôt que comme une prédiction absolue. Pour un facteur, le point de référence est sa catégorie de référence; pour une variable numérique, c’est sa moyenne. Le cours avertit que transformer un contraste sur l’échelle de réponse (`scale = "response"`) peut être délicat à interpréter. [Voir la visualisation du modèle](GLM.html#visualize-the-model).

### Comparaison de modèles

Le test ANOVA (analyse de déviance) compare des modèles emboîtés ajustés aux mêmes données. Notons $D_0$ la déviance du modèle simple et $D_1$ celle du modèle plus complexe, qui contient le modèle simple comme cas particulier. Le modèle plus complexe s’ajustant au moins aussi bien, $\Delta D = D_0 - D_1 \geq 0$ (la différence est généralement strictement positive). Sous l’hypothèse nulle, $\Delta D$ suit approximativement une loi du Khi-deux avec $\Delta df$ degrés de liberté ajoutés. Au seuil $\alpha$, on rejette le modèle simple si :

$$\Delta D > \chi^2_{1-\alpha,\,\Delta df}.$$

`anova(m0, m1, test = "Chisq")` réalise ce test du rapport de vraisemblance. Si la p-valeur est petite ($p < \alpha$), on rejette le modèle simple et le gain justifie le modèle complexe. Si elle est grande ($p \geq \alpha$), les données ne fournissent pas assez d’évidence pour justifier la complexité supplémentaire; on conserve le modèle simple par parcimonie. [Voir la comparaison des modèles](GLM.html#comparing-two-models).

Rappel théorique : la déviance d’un modèle estimé est le double de la différence entre la log-vraisemblance du full model (modèle saturé) et celle du modèle estimé. En notant $L$ la log-vraisemblance :

$$D = 2\left[L_{\text{full}} - L(\widehat{\beta}, \psi)\right] = 2\left[\ell(\text{modèle saturé}) - \ell(M)\right].$$

À données fixées, $L_{\text{full}}$ est constant : maximiser la log-vraisemblance du modèle estimé revient donc à minimiser sa déviance.

Le modèle saturé reproduit les réponses observées ($\widehat{\mu}_i = y_i$) et sa déviance vaut zéro.

Dans le modèle saturé (`L_full`), chaque observation $i$ a son propre paramètre naturel $\widetilde{\theta}_i$, choisi de sorte que :

$$a'(\widetilde{\theta}_i) = y_i.$$

La moyenne ajustée reproduit alors la valeur observée, $\widehat{\mu}_i = y_i$.

La déviance scaled est la différence de log-vraisemblance entre le modèle saturé (parfois appelé « full model ») et le modèle ajusté, divisée par le paramètre de dispersion $\phi$ :

$$D_{\text{scaled}}(M) = \frac{D(M)}{\phi} = \frac{2}{\phi}\left[\ell(\text{modèle saturé}) - \ell(M)\right].$$

Pour la loi de Poisson, $\phi = 1$ : la déviance scaled est donc égale à la déviance. Ici, « full model » désigne le modèle saturé, et non le modèle `m1` qui comprend toutes les variables explicatives étudiées.

Le modèle nul du cours comprend l’intercept et l’offset (log-exposition), sans variable explicative. La déviance nulle et la déviance résiduelle du modèle ajusté sont donc :

$$D_{\text{null}} = 2\left[\ell(\text{saturé}) - \ell(\text{nul})\right], \qquad D_{\text{résiduelle}} = 2\left[\ell(\text{saturé}) - \ell(\text{modèle ajusté})\right].$$

Pour une réponse de Poisson, la contribution de l’observation $i$ est :

$$D_i = 2\left[y_i \log\left(\frac{y_i}{\widehat{\mu}_i}\right) - (y_i - \widehat{\mu}_i)\right],$$

avec $y_i\log(y_i/\widehat{\mu}_i)=0$ si $y_i=0$. La déviance résiduelle est la somme de ces contributions : $D_{\text{résiduelle}}=\sum_i D_i$.

Pour des modèles emboîtés, la déviance résiduelle mesure l’écart au modèle saturé : elle diminue à mesure que l’ajustement s’améliore, jusqu’à zéro pour le modèle saturé.

Le choix d’un modèle entre le modèle nul et un modèle plus riche cherche un compromis entre qualité d’ajustement et complexité. La diminution de déviance mesure le gain d’ajustement, à évaluer par rapport aux degrés de liberté ajoutés (par exemple avec le test ANOVA du rapport de vraisemblance); la déviance résiduelle seule favorise généralement les modèles plus complexes.

### Prédictions

Pour prédire sur l’échelle de la réponse, utiliser `predict(m1, newdata, type = "response")`. Avec le lien logarithmique et l’offset d’exposition, cela renvoie la moyenne prédite du nombre de sinistres : $\widehat{\mu}_i = d_i \exp(\widehat{\text{score}}_i)$. Pour obtenir la fréquence par unité d’exposition, diviser cette prédiction par $d_i$. L’argument `type = "link"` (et non `"linear"`) renvoie le prédicteur linéaire, offset compris, plutôt qu’une prédiction sur l’échelle de la réponse. [Voir la prédiction des fréquences](GLM.html#predict-claim-frequencies).

### Modèle de taux et pondération

Si l’on modélise directement le taux par observation, $\text{ClaimNb}_i / d_i$, il faut pondérer chaque observation par son exposition $d_i$ afin de tenir compte de la durée d’exposition : `glm(ClaimNb / Exposure ~ ..., data = training_set, weights = Exposure, family = poisson(link = "log"))`. Dans l’exemple du cours, cette formulation donne les mêmes coefficients que le modèle des nombres de sinistres avec `offset(log(Exposure))`. [Voir la section Offset or weights](GLM.html#offset-or-weights).
