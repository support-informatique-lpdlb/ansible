# ansible-aos8

Onboarding des switchs Alcatel-Lucent Enterprise OmniSwitch (AOS 8), collection
`ale.aos8`, connexion SSH (`network_cli`), secrets Ansible Vault, inventaire par site.

Le playbook `onboard.yml` envoie la config de base d'un switch en SSH
(`aos8_command`) : nom système, services IP, AAA, VLANs, agrégats LACP, comptes
locaux — puis sauvegarde et certifie.

`--check` = **dry-run sûr** : il affiche seulement les commandes, sans rien écrire
(`aos8_command` n'exécute pas les commandes de config en check mode). Un run réel
(ré)applique tout ; sur AOS c'est sans danger. `aos8_config` a été écarté : sur
cette collection il applique même en `--check` et son diff n'est pas fiable.

## Installation (nœud de contrôle Linux)

```bash
sudo apt install -y pipx && pipx ensurepath
pipx install "ansible-core>=2.14"
pipx inject ansible-core paramiko
ansible-galaxy collection install -r requirements.yml -p collections
```

> `ale.aos8` n'est pas sur la Galaxy publique : `requirements.yml` l'installe
> depuis l'archive GitHub (v1.0.2, dernière release stable).
>
> Auth par mot de passe : `ansible.cfg` force `ssh_type = paramiko`
> (`look_for_keys = False`) pour ne pas tenter de clé SSH sur les switchs.

## Structure

```
inventories/
  ROU/hosts.yml                  # switchs du site ROU
  group_vars/ROU/main.yml        # connexion + management + plan de VLANs
  group_vars/ROU/vault.yml       # secrets chiffrés (modèle : vault.yml.example)
  host_vars/<switch>.yml         # overrides config/ config supplémentaire
playbooks/onboard.yml            # onboarding (aos8_command ; --check = dry-run sûr)
templates/onboard.cfg.j2         # config AOS 8 rendue depuis les variables
```

Portée des variables : **site** (`group_vars/<SITE>/`) = connexion, mgmt, VLANs +
secrets ; **switch** (`host_vars/<switch>.yml`) = `linkaggs`.

## Vault

```bash
cd inventories/group_vars/ROU
cp vault.yml.example vault.yml     # renseigner les valeurs
ansible-vault encrypt vault.yml    # (ansible-vault edit vault.yml pour modifier)
```

Variables : `vault_aos8_user` / `vault_aos8_password` (connexion SSH),
`vault_user_admin_password` / `vault_user_adminacs_password` (comptes posés par le
playbook), `vault_radius_key` (clé partagée RADIUS/ISE).

## Lancer sur un switch

1. Déclarer le switch dans `inventories/ROU/hosts.yml` (nom + `ansible_host`).
2. (Optionnel) `host_vars/<switch>.yml` pour ses `linkaggs`.
3. Toujours cibler avec `--limit` :

```bash
# Vérifier la cible + la connexion
ansible-inventory --host LAB_TEST_01
ansible LAB_TEST_01 -m ale.aos8.aos8_facts --ask-vault-pass

# Dry-run : affiche les commandes, n'écrit rien
ansible-playbook playbooks/onboard.yml --limit LAB_TEST_01 --check --ask-vault-pass

# Application
ansible-playbook playbooks/onboard.yml --limit LAB_TEST_01 --ask-vault-pass
```

Le reload est désactivé par défaut (`-e onboard_reload=true` pour rebooter en fin de run).

## Ce que pousse `onboard.yml`

`auto-fabric disable`, `spantree mode flat`, `system name`, `session prompt`,
`ip service`, `aaa`, `vlan 1` off + `mgmt_vlan` on, nommage des VLANs du site,
agrégats LACP + membership (trunk = tous les VLANs sauf `uplink_exclude_vlans`),
**RADIUS/ISE** (serveurs `radius_servers`, profil AAA `802.1x` + accounting, profils
UNP `unp_profiles` → VLAN, port-template `unp_port_template`), comptes `admin` / `admin-acs`.

Interface `MGMT` + route par défaut : **différé** (en commentaire dans le template).
Auth admin du switch via ISE : optionnelle (`admin_auth_via_ise: true`).

## Ajouter un site

Créer `inventories/<SITE>/hosts.yml` + `group_vars/<SITE>/{main.yml,vault.yml}`
(repartir de ROU), puis cibler `--limit <SITE>`.

## host_vars — exemple

```yaml
linkaggs:
  - { id: 32, name: ROU-SW-ACCESS-01, ports: [1/1/48, 2/1/48], trunk: true }
  - { id: 31, name: NAS-BACKUP,       ports: [1/1/7, 2/1/47], vlans_untagged: [15] }
```

`trunk: true` = tous les VLANs du site sauf `uplink_exclude_vlans`
(`group_vars/ROU/main.yml`, défaut `[2, 123]`). Sinon `vlans_tagged` / `vlans_untagged`.

## AWX

`execution-environment/` : image `ansible-builder` (avec `ale.aos8`) pour AWX.
