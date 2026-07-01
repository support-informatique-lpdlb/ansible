# ansible-aos8

Onboarding des OmniSwitch Alcatel-Lucent Enterprise (AOS 8) via la collection
`ale.aos8` — connexion `network_cli` (SSH), secrets Ansible Vault, inventaire
multi-sites.

Le playbook `onboard.yml` applique la configuration de base d'un switch de façon
**idempotente** (`aos8_config` compare au running-config et n'envoie que ce qui
manque) : nom système, services IP, AAA, plan de VLANs, agrégats LACP et comptes
locaux — puis sauvegarde et certifie si des changements ont eu lieu. Chaque
**site** porte ses propres variables (connexion, secrets, VLANs).

## Prérequis

Nœud de contrôle Linux (Ubuntu / WSL) :

```bash
python3 -m pip install --user "ansible-core>=2.14" ansible-pylibssh
ansible-galaxy collection install -r requirements.yml -p collections
```

## Structure

```
ansible.cfg                        # inventory = inventories/, collections_path
requirements.yml                   # ale.aos8, ansible.netcommon, ansible.utils
inventories/                       # inventaire = ce répertoire (un dossier par site)
  group_vars/
    ROU/main.yml                   # site ROU : connexion + management + plan de VLANs
    ROU/vault.yml                  # site ROU : secrets chiffrés (voir vault.example.yml)
  host_vars/
    ROU_SW_ACCESS_02.yml           # par switch : agrégats LACP (+ mgmt_ip pour plus tard)
  ROU/hosts.yml                    # switchs du site ROU (un seul groupe)
playbooks/onboard.yml              # onboarding d'un switch (aos8_config, idempotent)
templates/onboard.cfg.j2           # config AOS 8 rendue depuis les variables
```

## Inventaire (multi-sites)

L'inventaire est le répertoire `inventories/` ; chaque site a son dossier
(`inventories/<SITE>/hosts.yml`) et **porte toutes ses variables** dans
`group_vars/<SITE>/` (pas de group_vars partagé) :

| Portée | Fichier                        | Contenu                                                  |
|--------|--------------------------------|----------------------------------------------------------|
| Site   | `group_vars/<SITE>/main.yml`   | connexion, `mgmt_vlan/mask/gateway`, `uplink_exclude_vlans`, plan de VLANs |
| Secrets| `group_vars/<SITE>/vault.yml`  | identifiants + mots de passe (chiffré)                   |
| Switch | `host_vars/<switch>.yml`       | `linkaggs` (+ `mgmt_ip` pour plus tard)                  |

```
aos8
└── ROU    : ROU_SW_CDR_01, ROU_SW_ACCESS_01, ROU_SW_ACCESS_02, ROU_SW_ACCESS_03
```

### Ajouter un site

1. Créer `inventories/<SITE>/hosts.yml` (`aos8 > <SITE> > hosts:`).
2. Créer `inventories/group_vars/<SITE>/main.yml` (connexion + management + `aos8_vlans`)
   et `group_vars/<SITE>/vault.yml` (secrets). Repartir du site ROU comme modèle.
3. Cibler avec `--limit <SITE>`.

## Vault

```bash
cd inventories/group_vars/ROU
cp vault.example.yml vault.yml          # renseigner les valeurs
ansible-vault encrypt vault.yml
```

Mot de passe fourni au lancement via `--ask-vault-pass`.

## Onboarder un switch

1. **Déclarer le switch** dans `inventories/<SITE>/hosts.yml` (nom + `ansible_host`).
2. **Créer son `host_vars/<switch>.yml`** avec les `linkaggs` (voir
   `host_vars/ROU_SW_ACCESS_02.yml`). `mgmt_ip` y est déjà prévu pour plus tard.
3. **Tester puis appliquer**, toujours ciblé sur ce switch :

```bash
# Simulation : montre les lignes qui seraient appliquées (diff vs running-config)
ansible-playbook playbooks/onboard.yml --limit ROU_SW_ACCESS_02 --check --diff --ask-vault-pass

# Application (n'envoie que ce qui manque ; ne sauvegarde que si ça a changé)
ansible-playbook playbooks/onboard.yml --limit ROU_SW_ACCESS_02 --ask-vault-pass

# Avec reload du switch en fin de run (coupe la session)
ansible-playbook playbooks/onboard.yml --limit ROU_SW_ACCESS_02 -e onboard_reload=true --ask-vault-pass
```

> Sur un switch neuf, la connexion se fait avec les identifiants usine
> (`vault_aos8_user` / `vault_aos8_password`, ex. `admin`/`switch`) ; le playbook
> positionne ensuite les mots de passe définitifs (`vault_user_*`).

Vérifier la cible avant exécution :

```bash
ansible ROU --list-hosts
ansible-inventory --host ROU_SW_ACCESS_02
```

## Ce que fait `onboard.yml`

Rend `templates/onboard.cfg.j2` depuis les variables et l'applique via
**`aos8_config`** : la config désirée est comparée au running-config, **seules les
lignes manquantes sont envoyées** (idempotent). `write memory flash-synchro` +
`copy running certified` uniquement si des changements ont eu lieu :

- `auto-fabric disable`, `spantree mode flat`, `system name`, `session prompt`
- `ip service` (all + ssh), `aaa authentication` (local / http / ssh)
- `vlan 1` désactivé, `mgmt_vlan` activé
- création + nommage de tous les VLANs du site (`aos8_vlans`)
- agrégats LACP (`linkaggs`) + affectation des VLANs (trunk = tous sauf
  `uplink_exclude_vlans`)
- comptes locaux `admin-acs` / `admin` (mots de passe depuis le vault)
- interface `MGMT` + route par défaut : **différé** (lignes en commentaire dans le template)

> Un 2ᵉ run sur un switch déjà configuré ne renvoie rien (`changed=false`) et ne
> re-sauvegarde pas. Idéal pour rejouer l'onboarding sans risque.

## Variables clés

`host_vars/<switch>.yml` :

```yaml
mgmt_ip: 10.2.121.102
linkaggs:
  - { id: 32, name: ROU-SW-ACCESS-01, ports: [1/1/48, 2/1/48], trunk: true }
  - { id: 30, name: ROU-SW-ACCESS-03, ports: [1/1/49, 2/1/49], trunk: true }
  - { id: 31, name: NAS-BACKUP,       ports: [1/1/7, 2/1/47], vlans_untagged: [15] }
```

`trunk: true` tague tous les VLANs du site sauf `uplink_exclude_vlans`
(défini dans `group_vars/ROU/main.yml`, défaut `[2, 123]`). Sinon utiliser
`vlans_tagged` / `vlans_untagged` explicites.

## AWX

`execution-environment/` : image `ansible-builder` (contenant `ale.aos8`) pour
l'exécution via AWX.
