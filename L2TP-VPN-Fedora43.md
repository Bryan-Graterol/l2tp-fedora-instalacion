# L2TP/IPsec VPN en Fedora 43 — Guía de instalación limpia

> **Verificado:** Fedora 43 · NetworkManager-l2tp 1.52.0 · libreswan (pluto/IKEv1)  
> **Estado:** Conexión exitosa confirmada ✓

---

## Contexto

Fedora 43 incluye tres cambios que rompen L2TP/IPsec por defecto:

| Problema | Causa | Solución |
|---|---|---|
| `ikev1-policy=drop` | libreswan ≥ 5.0 deshabilita IKEv1 | Descomentar línea en `/etc/ipsec.conf` |
| Módulos L2TP inactivos | `kernel-modules-extra` los blacklistea | Comentar las líneas en modprobe.d |
| `xl2tpd` no disponible | NM-l2tp 1.52+ usa su propio demonio | No se necesita instalar |

---

## 1. Instalación de paquetes

```bash
sudo dnf install -y \
    NetworkManager-l2tp \
    NetworkManager-l2tp-gnome \
    libreswan \
    --skip-unavailable
```

> `xl2tpd` **no** se instala — NetworkManager-l2tp 1.52+ incluye su propio demonio L2TP interno (`nm-l2tp-service`) y usa `pppd` directamente vía `/usr/lib64/pppd/2.5.1/nm-l2tp-pppd-plugin.so`.

Verificar versiones:

```bash
rpm -q NetworkManager-l2tp libreswan
```

---

## 2. Desbloquear módulos L2TP del kernel

Los módulos L2TP están en el paquete `kernel-modules-extra` y vienen **blacklisteados por defecto**.

```bash
sudo sed -e '/blacklist l2tp_netlink/s/^b/#b/g' -i /etc/modprobe.d/l2tp_netlink-blacklist.conf
sudo sed -e '/blacklist l2tp_ppp/s/^b/#b/g' -i /etc/modprobe.d/l2tp_ppp-blacklist.conf
```

Verificar que quedaron comentados:

```bash
grep -E 'l2tp' /etc/modprobe.d/l2tp_netlink-blacklist.conf /etc/modprobe.d/l2tp_ppp-blacklist.conf
# Resultado esperado: líneas con #blacklist (comentadas)
```

Cargar los módulos en la sesión actual:

```bash
sudo modprobe l2tp_ppp l2tp_netlink l2tp_core
lsmod | grep l2tp
```

Para persistencia entre reinicios (opcional, los archivos blacklist editados ya lo garantizan en la mayoría de casos):

```bash
sudo tee /etc/modules-load.d/l2tp.conf > /dev/null <<'EOF'
l2tp_ppp
l2tp_netlink
l2tp_core
EOF
```

---

## 3. Habilitar IKEv1 en libreswan

libreswan ≥ 5.0 deshabilita IKEv1 globalmente. L2TP/IPsec lo requiere obligatoriamente.

```bash
sudo sed -e 's/#ikev1-policy=.*/ikev1-policy=accept/' -i /etc/ipsec.conf
```

Verificar:

```bash
grep ikev1-policy /etc/ipsec.conf
# Resultado esperado: ikev1-policy=accept  (sin #)
```

---

## 4. Inicializar base de datos NSS (si es instalación nueva)

```bash
sudo ipsec initnss 2>/dev/null || echo "NSS ya inicializado"
```

---

## 5. Habilitar y reiniciar servicios

```bash
sudo systemctl enable --now ipsec
sudo systemctl restart NetworkManager
```

Verificar que libreswan arrancó sin errores de IKEv1:

```bash
sudo journalctl -u ipsec --since "1 min ago" --no-pager | grep -E 'ikev1|error|failed|drop'
# No debe aparecer: "ikev1-policy=drop does not allow IKEv1 connections"
```

Warnings **inofensivos** que pueden aparecer (no bloquean el funcionamiento):

```
Symbol `ldns_error_str' has different size in shared object  → desincronización de versión ldns, ignorar
IPTFS ipsec SA error: requires option CONFIG_XFRM_IPTFS     → feature opcional del kernel, ignorar
```

---

## 6. Crear la conexión VPN

### Opción A — Interfaz gráfica (GNOME)

**Configuración → Red → VPN → `+` → Layer 2 Tunneling Protocol (L2TP)**

Campos requeridos:
- **Gateway:** IP o hostname del servidor VPN
- **Username / Password:** credenciales PPP
- En **IPsec Settings...** → activar IPsec y configurar la clave precompartida (PSK)

### Opción B — nmcli

```bash
nmcli connection add \
    type vpn \
    vpn-type l2tp \
    con-name "NombreVPN" \
    vpn.data "gateway=IP_SERVIDOR, user=USUARIO, password-flags=1, ipsec-enabled=yes, ipsec-psk=CLAVE_PSK"
```

---

## 7. Verificación de conexión exitosa

Al conectar correctamente, el journal mostrará:

```
pluto: IKEv1 Main Mode initiated
pluto: STATE_MAIN_I4: ISAKMP SA established
pppd: CHAP authentication succeeded
pppd: local  IP address X.X.X.X
pppd: remote IP address X.X.X.X
NetworkManager: policy: set 'NombreVPN' (ppp0) as default for IPv4 routing and DNS
```

Monitor en tiempo real al conectar:

```bash
sudo journalctl -u ipsec -u NetworkManager -f
```

---

## Diagnóstico rápido

| Síntoma | Causa probable | Verificación |
|---|---|---|
| `ikev1-policy=drop does not allow IKEv1` | Paso 3 no aplicado | `grep ikev1-policy /etc/ipsec.conf` |
| `Could not add ipsec connection` | IKEv1 bloqueado o módulos faltantes | `lsmod \| grep l2tp` |
| PPP no negocia | Módulos no cargados | `lsmod \| grep l2tp_ppp` |
| Pluto no arranca | NSS no inicializado | `sudo ipsec initnss` |
| Denials SELinux | SELinux en enforcing | `sudo ausearch -m avc -ts today \| grep l2tp` |

Para diagnóstico SELinux si hay denials:

```bash
sudo ausearch -m avc -ts today | grep -E 'ipsec|pluto|l2tp|ppp' | audit2allow -M nm-l2tp-local
sudo semodule -i nm-l2tp-local.pp
```

---

## Resumen del orden crítico

```
1. dnf install NetworkManager-l2tp libreswan
2. Comentar blacklists en /etc/modprobe.d/
3. ikev1-policy=accept en /etc/ipsec.conf
4. modprobe l2tp_ppp l2tp_netlink l2tp_core
5. systemctl enable --now ipsec
6. systemctl restart NetworkManager
7. Conectar VPN
```
