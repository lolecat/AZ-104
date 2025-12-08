Lab où on va créer :
- 10 utilisateurs, répartis en 3 groupes :
  - 2 utilisateurs Admins
  - 4 utilisateurs Groupe A
  - 4 utilisateurs Groupe B

- 2 Resource Groups :
  - Un pour en dev 
  - Un pour env prod

- Les utilisateurs du groupe A ont les droits sur prod (RBAC)
- Les utilisateurs du groupe B ont les droits sur dev (RBAC)

- Utiliser une policy pour l'heritage des tags :
  - "env = prod","team = A" pour RG prod
  - "env = prod","team = B" pour RG dev