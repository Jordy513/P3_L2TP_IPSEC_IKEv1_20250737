# Capturas de pantalla — VPN Client-to-Site L2TP sobre IPSec (IKEv1)

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_config_r2_isakmp.png`](/screenshots/02_config_r2_isakmp.png) | Consola de R2 mostrando la `crypto isakmp policy 10` con 3DES/SHA/PSK y el `crypto isakmp key` con wildcard `0.0.0.0`. |
| 3 | [`03_config_r2_vpdn.png`](/screenshots/03_config_r2_vpdn.png) | Consola de R2 mostrando el `vpdn-group L2TP-CLIENTES`, el `Virtual-Template1` y el `ip local pool POOL-VPN`. |
| 4 | [`04_cliente_win7_config_vpn.png`](/screenshots/04_cliente_win7_config_vpn.png) | Ventana de propiedades de la conexión VPN en Windows 7 mostrando tipo L2TP/IPSec y la clave precompartida `MiClaveVPN123` configurada. |
| 5 | [`05_isakmp_sa_qmidle.png`](/screenshots/05_isakmp_sa_qmidle.png) | Salida de `show crypto isakmp sa` en R2 mostrando estado `QM_IDLE` con la IP del cliente. |
| 6 | [`06_vpdn_session_est.png`](/screenshots/06_vpdn_session_est.png) | Salida de `show vpdn session` en R2 mostrando `cliente1` con estado `est` y la interfaz `Vi1`. |
| 7 | [`07_pool_in_use.png`](/screenshots/07_pool_in_use.png) | Salida de `show ip local pool POOL-VPN` mostrando `In use: 1` — IP asignada al cliente. |
| 8 | [`08_win7_ipconfig_vpn.png`](/screenshots/08_win7_ipconfig_vpn.png) | Salida de `ipconfig /all` en Windows 7 mostrando el adaptador VPN con IP asignada `20.25.37.100` del pool del servidor. |
| 9 | [`09_ping_red_interna.png`](/screenshots/09_ping_red_interna.png) | Ping exitoso desde el Cliente Windows 7 (cmd) hacia PC1 (`20.25.37.2`) con la VPN activa, TTL=63. |
