Lab où on va créer :

#### Céation d'utilisateurs et de groupes, gestion des accès admin
- 10 utilisateurs, répartis en 3 groupes :
  - 1 utilisateur "Super Admin"
  - 1 utilisateur "Subscriptions Manager"
  - 4 utilisateurs Equipe A
  - 4 utilisateurs Equipe B

#### Gestion des Abonnements
- 2 Subscriptions :
  - Une pour Equipe A
  - Une pour Equipe B

#### Gestion des Management Groups + RBAC
- 2 MG dans chaque Subscription :
  - Dev
  - Prod

- Les utilisateurs Equipe A ont les droits sur Sub A / MG A (RBAC)
- Les utilisateurs Equipe B ont les droits sur Sub B / MG B (RBAC)

#### Policies
- Utiliser une policy pour l'heritage des tags :
  - "env = prod","team = A" pour RG prod
  - "env = prod","team = B" pour RG dev