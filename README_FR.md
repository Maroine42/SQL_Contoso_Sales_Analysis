# Analyse des Ventes Contoso

## Vue d’ensemble

Bienvenue dans mon analyse du comportement client, de la rétention et de la valeur client à long terme pour une entreprise de e-commerce, dans le but d’améliorer la fidélisation et de maximiser les revenus.

Pour ce projet, j’utilise le jeu de données Contoso, une base de données d'exemple fournie par Microsoft qui simule les données d'une entreprise multinationale fictive, Contoso Ltd. Ce dataset est largement utilisé pour l’apprentissage et la démonstration de compétences en SQL, en business intelligence et en analyse de données.

Il contient un large éventail de données métier réalistes, notamment :

- Ventes : commandes, clients, zones géographiques, points de vente

- Produits : catégories, sous-catégories, informations produits

- Finance : revenus, coûts, bénéfices

- Temps : dimensions temporelles (date et heure) utiles pour le reporting

Ce dataset me permet de travailler avec des requêtes complexes, d'explorer des indicateurs business, et de transformer des données brutes en analyses pertinentes.

## Problématiques Métier

1. Segmentation client : Qui sont nos clients les plus précieux ?

2. Analyse par cohorte : Comment les différents groupes de clients génèrent-ils du chiffre d'affaires ?

3. Analyse de rétention : Quels sont les clients qui n’ont pas effectué d’achat récemment ?

## Outils Utilisés

1. PostgreSQL : Utilisé comme système principal de gestion de base de données pour héberger le dataset Contoso.

2. DBeaver : Utilisé comme client SQL pour l’écriture et l’exécution des requêtes.

3. Visual Studio Code : Servi à rédiger les scripts SQL et documenter le projet.

4. Git : Utilisé pour le versioning et le suivi des modifications tout au long du projet.

# La Vue SQL

```sql
CREATE VIEW cleaned_customer AS
WITH cohort_data AS (
    SELECT 
        s.customerkey,
        c.countryfull, 
        c.age,
        c.givenname,
        c.surname,
        MIN(s.orderdate) OVER (PARTITION BY s.customerkey) AS first_purchase_date,
        EXTRACT(YEAR FROM MIN(s.orderdate) OVER (PARTITION BY s.customerkey)) AS cohort_year,
        s.orderdate,
        COUNT(s.orderkey) AS num_orders,
        SUM(s.quantity* s.netprice * s.exchangerate) AS total_net_revenue
    FROM sales s
    LEFT JOIN customer c ON c.customerkey = s.customerkey
    GROUP BY 
        s.customerkey, 
        c.countryfull, 
        c.age, 
        c.givenname, 
        c.surname,
        s.orderdate
)

SELECT
    cd.customerkey, 
    cd.cohort_year,
    CONCAT(TRIM(cd.givenname), ' ', TRIM(cd.surname)) AS cleaned_name, 
    COALESCE(cd.num_orders, 0) AS num_orders,
    COALESCE(cd.total_net_revenue, 0) AS net_revenue,
    cd.countryfull,
    cd.age,
    cd.first_purchase_date,
    cd.orderdate
FROM cohort_data cd;
```

## Importance des Vues en SQL

1. Abstraction & Simplification

Les vues permettent de masquer une logique SQL complexe derrière une interface simple. Plutôt que de répéter des jointures, calculs ou fonctions de fenêtrage à chaque requête, on crée une vue une seule fois et on l’interroge comme une table.

2. Réutilisabilité du Code

Une vue bien conçue, comme cleaned_customer, peut être réutilisée dans plusieurs rapports, tableaux de bord ou analyses. Elle centralise la logique (analyse par cohorte, calcul de revenus, etc.) et permet de garder un code DRY (Don’t Repeat Yourself).

3. Sécurité et Contrôle d’Accès

Les vues permettent de limiter l’accès à certaines colonnes ou lignes. Cela permet de protéger les données sensibles tout en offrant des possibilités d’analyse. Par exemple, si une table clients contient des identifiants confidentiels, une vue peut les masquer.

4. Cohérence des Données

Les vues garantissent une logique uniforme. Par exemple, si la formule de calcul du revenu est SUM(quantity * netprice * exchangerate), elle sera toujours appliquée de la même manière partout où la vue est utilisée.

5. Optimisation des Performances

Bien que les vues ne soient pas toujours plus rapides, elles peuvent l’être si on utilise des materialized views (vues matérialisées), lorsque cela est possible. Même les vues classiques peuvent aider l’optimiseur de requêtes en simplifiant la logique.

6. Facilité de Maintenance

Quand la logique métier évolue (ex. : modification du calcul du revenu net), il suffit de mettre à jour la vue une seule fois pour que toutes les requêtes dépendantes soient automatiquement à jour. C’est un vrai gain en maintenance à long terme.


## Démarche d’Analyse

## 1. Analyse de la Segmentation Client

- Classement des clients selon leur LTV (Lifetime Value – LTV)

- Attribution des clients à trois segments : valeur élevée, intermédiaire, faible

- Calcul des indicateurs clés : chiffre d’affaires total

## Requête SQL

```sql
WITH customer_ltv AS (
	SELECT
		customerkey,
		cleaned_name,
		SUM(total_net_revenue) AS total_ltv
	FROM cohort_analysis
	GROUP BY
		customerkey,
		cleaned_name
), customer_segments AS (
	SELECT
		PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total_ltv) AS ltv_25th_percentile,
		PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total_ltv) AS ltv_75th_percentile
	FROM customer_ltv 
), segment_values AS (
	SELECT
		c.*,
		CASE
			WHEN c.total_ltv < cs.ltv_25th_percentile THEN '1 - Low-Value'
			WHEN c.total_ltv <= cs.ltv_75th_percentile THEN '2- Mid-Value'
			ELSE '3 - High-Value'
		END AS customer_segment
	FROM customer_ltv c,
		customer_segments cs
)
SELECT
	customer_segment,
	SUM(total_ltv) AS total_ltv,
	COUNT(customerkey) AS customer_count,
	SUM(total_ltv) / COUNT(customerkey)
FROM segment_values
GROUP BY
	customer_segment
ORDER BY
	customer_segment DESC
```

## Visualisation

![Value Customers](images/1_customer_segementation.png)
*Segmentation Client par Valeur à Vie (LTV) : Répartition des Segments Élevé, Moyen et Faible.*

### Résultats Clés

- Le segment à haute valeur (25 % des clients) génère 66 % des revenus (135,4 M$)

- Le segment à valeur moyenne (50 % des clients) génère 32 % des revenus (66,6 M$)

- Le segment à faible valeur (25 % des clients) représente 2 % des revenus (4,3 M$)

### Informations Métier

- Haute Valeur (66 % des revenus) : Proposer un programme d'adhésion premium aux 12 372 clients VIP, car perdre un client a un impact significatif sur les revenus.

- Valeur Moyenne (32 % des revenus) : Créer des parcours de mise à niveau grâce à des promotions personnalisées, avec un potentiel de revenus allant de 66,6 M$ à 135,4 M$.

- Faible Valeur (2 % des revenus) : Concevoir des campagnes de réengagement et des promotions sensibles au prix pour augmenter la fréquence d'achat.

## 2. Analyse par Cohorte

- Suivi des revenus et du nombre de clients par cohorte

- Les cohortes ont été regroupées selon l'année du premier achat

- Analyse de la rétention des clients au niveau des cohortes

## Requête SQL

```sql
SELECT 
	cohort_year,
	COUNT(DISTINCT customerkey ) AS total_customers,
	SUM(total_net_revenue) AS total_revenue,
	SUM(total_net_revenue)/COUNT(DISTINCT customerkey) AS customer_revenue
FROM cohort_analysis
WHERE orderdate = first_purchase_date 
GROUP BY 
	cohort_year
```

## Visualisation

![Cohort Analysis](images/2_cohort_analysis.png)
*Tendances des Revenus Clients par Année de Cohorte (Ajustées en Fonction du Temps sur le Marché)*

### Résultats Clés

- Les revenus par client montrent une tendance alarmante de baisse au fil du temps

    - Les cohortes de 2022-2024 affichent des performances systématiquement inférieures à celles des cohortes précédentes

    - Remarque : Bien que les revenus nets augmentent, cela est probablement dû à une base de clients plus large, ce qui ne reflète pas nécessairement la valeur des clients.

### Informations Métier

- La valeur extraite des clients diminue avec le temps et nécessite une investigation plus approfondie.

- En 2023, nous avons observé une baisse du nombre de clients acquis, ce qui est préoccupant.

- Avec une baisse de la LTV et une diminution de l'acquisition de clients, l'entreprise pourrait être confrontée à un déclin des revenus.

## 3. Rétention Client

## Requête SQL

```sql
WITH customer_last_purchase AS (
	SELECT
		customerkey,
		cleaned_name,
		orderdate,
		ROW_NUMBER() OVER (PARTITION BY customerkey ORDER BY orderdate DESC) AS rn,
		first_purchase_date,
		cohort_year
	FROM
		cohort_analysis
), inactive_customers AS (
	SELECT
		customerkey,
		cleaned_name,
		orderdate AS last_purchase_date,
		CASE
			WHEN orderdate < (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months' THEN 'Churned'
			ELSE 'Active'
		END AS customer_status,
		cohort_year
	FROM customer_last_purchase 
	WHERE rn = 1
		AND first_purchase_date < (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months'
)
SELECT
	cohort_year,
	customer_status,
	COUNT(customerkey) AS num_customers,
	SUM(COUNT(customerkey)) OVER(PARTITION BY cohort_year) AS total_customers,
	ROUND(COUNT(customerkey) / SUM(COUNT(customerkey)) OVER(PARTITION BY cohort_year), 2) AS status_percentage
FROM inactive_customers 
GROUP BY cohort_year, customer_status
```

- Identification des clients à risque d'inactivité

- Analyse des comportements d'achat récents

- Calcul des indicateurs spécifiques aux clients : Tendances de la rétention client vs. inactivité par année de cohorte

## Visualisation

![Retention](images/3_customer_churn_cohort_year.png)
*Tendances de la Rétention Client vs. Inactivité par Année de Cohorte*

### Résultats Clés

- Le taux de d'inactivité des cohortes se stabilise autour de 90 % après 2-3 ans, ce qui indique un modèle de rétention à long terme prévisible.

- Les taux de rétention sont systématiquement faibles (8-10 %) pour toutes les cohortes, suggérant que les problèmes de rétention sont structurels et non spécifiques à certaines années.

- Les cohortes récentes (2022-2023) montrent des trajectoires de désabonnement similaires, ce qui indique qu’en l'absence d’intervention, les futures cohortes suivront le même modèle.

### Informations Métier

- Renforcer les stratégies d’engagement précoce pour cibler les 1-2 premières années avec des incitations à l’intégration, des récompenses de fidélité et des offres personnalisées afin d’améliorer la rétention à long terme.

- Réengager les clients à forte valeur ayant abandonné en se concentrant sur des campagnes ciblées de reconquête plutôt que sur des efforts de rétention généraux, car réactiver des utilisateurs précieux peut offrir un meilleur retour sur investissement (ROI).

- Prédire et prévenir le risque de d'inactivité en utilisant des indicateurs d'alerte spécifiques aux clients pour intervenir de manière proactive auprès des utilisateurs à risque avant qu’ils ne se désabonnent.

# Recommandations Stratégiques

1. Optimisation de la Valeur Client (Segmentation Client)

    - Lancer un programme VIP pour 12 372 clients à forte valeur (66 % des revenus).

    - Créer des parcours de mise à niveau personnalisés pour le segment de valeur moyenne (potentiel de revenus de 66,6 M$ → 135,4 M$).

    - Concevoir des promotions sensibles au prix pour le segment à faible valeur afin d’augmenter la fréquence d'achat.

2. Stratégie de Performance des Cohortes (Revenus Clients par Cohorte)

    - Cibler les cohortes 2022-2024 avec des offres personnalisées de réengagement.

    - Mettre en place des programmes de fidélité/abonnement pour stabiliser les fluctuations des revenus.

    - Appliquer les stratégies réussies des cohortes 2016-2018 à forte dépense aux nouveaux clients.

3. Rétention & Prévention du Désabonnement (Rétention Client)

    - Renforcer l’engagement des 1-2 premières années avec des incitations à l’intégration et des récompenses de fidélité.

    - Se concentrer sur des campagnes ciblées de reconquête pour les clients à forte valeur ayant abandonné.

    - Mettre en place un système d’intervention proactive pour les clients à risque avant qu’ils ne se désabonnent.

# Ce que J'ai Appris

1. Création de Requêtes Avancées & Modélisation de Données

J'ai renforcé ma capacité à rédiger des requêtes SQL complexes en utilisant des CTE (Common Table Expressions), des fonctions de fenêtrage et des agrégations. La création de vues réutilisables comme cleaned_customer montre également ma compréhension de la modélisation des données et de la manière de structurer les requêtes pour plus de clarté et de réutilisabilité.

2. Pensée Analytique dans un Contexte Métier

J'ai appris à traduire des questions métier en logique SQL, comme identifier les clients ayant abandonné ou calculer les segments clients par LTV. Cela démontre ma capacité à transformer des données brutes en informations exploitables, une compétence clé pour tout rôle d’analyste de données ou BI.

3. Techniques de Nettoyage & Cohérence des Données

J'ai pratiqué le nettoyage et la standardisation des données directement en SQL, comme la mise en forme des noms de clients, la gestion des valeurs NULL avec COALESCE, et l’assurance de calculs cohérents pour les revenus. Ces compétences sont essentielles pour garantir la qualité et la cohérence des données à travers les rapports et tableaux de bord.

# Défis Rencontrés

- Complexité des Données & Relations : Le dataset Contoso contient plusieurs tables interconnectées (ventes, clients, produits, finance, temps), ce qui nécessitait une compréhension approfondie des relations entre les tables et des clés étrangères pour joindre les données avec précision sans duplication ni perte d’informations.

- Gestion des Données Incomplètes ou Manquantes : Le dataset Contoso contient plusieurs tables interconnectées (ventes, clients, produits, finance, temps), ce qui nécessitait une compréhension approfondie des relations entre les tables et des clés étrangères pour joindre les données avec précision sans duplication ni perte d’informations.

- Traduire les Données en Informations Exploitables : Transformer les résultats des requêtes SQL en informations commerciales significatives a été un défi. Cela a exigé non seulement une exécution technique mais aussi la capacité d’interpréter les tendances et de proposer des stratégies basées sur les données pour la rétention des clients et la croissance des revenus.

# Conclusion 

Ce projet m'a permis d'appliquer des techniques SQL avancées pour analyser des défis commerciaux réels en utilisant le dataset Contoso. En explorant la segmentation client, le comportement des cohortes et les tendances de rétention, j’ai pu générer des informations exploitables pour soutenir la prise de décisions stratégiques autour de l’optimisation de la valeur client et de la prévention du désabonnement.

Grâce à cette analyse, j'ai approfondi mes compétences en modélisation de données, optimisation des requêtes et réflexion orientée métier. Plus important encore, j’ai renforcé l’importance d’un code SQL propre et bien structuré et son rôle dans la transformation de jeux de données complexes en récits significatifs qui favorisent la croissance de l’entreprise.

Cette expérience a non seulement renforcé ma base technique, mais elle a aussi amélioré ma capacité à réfléchir de manière critique et à communiquer efficacement des informations basées sur les données.







