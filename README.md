# bazzite-casa

Imagen OCI personalizada (BlueBuild) para los escritorios Bazzite de la familia
— issue #23, Fase 3. Hornea sobre **Bazzite GNOME nvidia-open** la pila de
identidad y control parental para que el rol Ansible `bazzite_ldap` (en el repo
`kxs-ansible`) solo tenga que **configurar**, no instalar:

- `sssd`, `sssd-ldap`, `sssd-tools` — login contra Authentik LDAP
- `oddjob`, `oddjob-mkhomedir` — creación de home en el primer login
- `autofs` — montaje bajo demanda de los homes NFS (Fase 4)
- `timekpr-next` (COPR) — control parental

Se publica en `ghcr.io/juanjocop/bazzite-casa:latest`, firmada con cosign.

## Estructura

```
recipes/recipe.yml          # la receta (base-image + módulos)
.github/workflows/build.yml # build diario + en cada push (blue-build/github-action@v1.11)
cosign.pub                  # clave pública de firma (commiteada)
cosign.key                  # clave PRIVADA — gitignored, NUNCA se commitea
files/system/               # ficheros a copiar en la imagen (vacío de momento)
```

## Puesta en marcha (una sola vez)

1. **Crea el repo en GitHub** y sube esto (ver "Primer push" abajo). El nombre del
   repo puede ser `bazzite-casa`.

2. **Añade el secret de firma.** En GitHub → Settings → Secrets and variables →
   Actions → New repository secret:
   - Nombre: `SIGNING_SECRET`
   - Valor: el contenido **íntegro** de `cosign.key` (incluye las líneas
     `-----BEGIN/END ENCRYPTED SIGSTORE PRIVATE KEY-----`).

   ```bash
   cat cosign.key   # copia TODO y pégalo como SIGNING_SECRET
   ```

   > La clave se generó **sin contraseña** (`COSIGN_PASSWORD=""`), que es lo que
   > espera la action. `cosign.key` está en `.gitignore`: no se sube nunca.

3. **Haz el package público (recomendado)** tras el primer build: GitHub → tu
   perfil → Packages → `bazzite-casa` → Package settings → Change visibility →
   Public. Si lo dejas privado, el host necesitará credenciales para rebasar.

4. **Lanza el primer build**: con el push ya arranca; o GitHub → Actions →
   workflow `bluebuild` → Run workflow. Tarda unos minutos.

## Conectar con kxs-ansible

En `kxs-ansible/inventory/group_vars/bazzite_desktops.yml`, sustituye el
`CHANGE_ME`:

```yaml
bazzite_image_ref: "ostree-image-signed:docker://ghcr.io/juanjocop/bazzite-casa:latest"
bazzite_image_match: "bazzite-casa"
```

## Rebasar el host (VM Bazzite) a esta imagen

⚠️ **El primer rebase NO puede verificar la firma** todavía: la política cosign
de *tu* registro vive DENTRO de esta imagen (módulo `signing`), y el host aún no
la tiene. Por eso el primer salto se hace `unverified` y, una vez arrancado en la
imagen (que ya trae la política), los updates siguientes sí verifican.

En la VM (consola noVNC o SSH como juanjocop):

```bash
# 1) primer rebase SIN verificar firma (la política aún no está en el host)
rpm-ostree rebase ostree-unverified-registry:ghcr.io/juanjocop/bazzite-casa:latest
systemctl reboot

# 2) ya arrancado en bazzite-casa, fija el rebase FIRMADO (verificado en adelante)
rpm-ostree rebase ostree-image-signed:docker://ghcr.io/juanjocop/bazzite-casa:latest
systemctl reboot
```

Tras esto, `/usr/sbin/sssd` y `/usr/sbin/oddjobd` existen y el preflight de
`bazzite_ldap` pasa. Enrola desde `kxs-ansible`:

```bash
ansible-playbook playbooks/bazzite_enroll.yml --limit sv1-bazzite --ask-vault-pass -K --diff
```

> El rol `bazzite_ldap` también tiene un bloque de rebase propio (`--tags rebase
> -e bazzite_allow_rebase=true`), pero usa `ostree-image-signed:` directamente, que
> falla en el PRIMER salto por lo de la firma. Para el estreno usa los comandos
> manuales de arriba; el bloque del rol sirve para mantener/forzar el rebase una
> vez la política ya está en el host.

## Primer push

```bash
cd bazzite-casa
git init -b main
git add .
git commit -m "esqueleto inicial bazzite-casa (SSSD/LDAP + autofs + timekpr)"
gh repo create bazzite-casa --public --source=. --remote=origin --push
# (o crea el repo a mano en github.com y: git remote add origin ... && git push -u origin main)
```
