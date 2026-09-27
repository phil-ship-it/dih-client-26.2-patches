# `provipvp:kit` für Minecraft 26.2

Die Kit-Funktion von ProviPvP brach beim Laden der Welt mit folgendem Fehler ab:

```
java.lang.IllegalArgumentException: Whilst parsing command on line 2:
Malformed 'minecraft:enchantments' component: 'Not a number; Failed to get element
minecraft:levels missed input: {levels:{"minecraft:mending":1,
"minecraft:sharpness":5,"minecraft:unbreaking":3}}'
```

## Ursache

In der Funktion standen die Verzauberungen als verschachtelter Block:

```mcfunction
item replace entity @p weapon.mainhand with minecraft:netherite_sword[
  minecraft:enchantments={levels:{"minecraft:sharpness":5,...}}
]
```

Der Parser kennt in 26.2 keinen `levels`-Schlüssel. Die Item-Komponente
`ItemEnchantments` ist in dieser Version als flache Map aufgebaut:

```java
CODEC = UnboundedMapCodec.unboundedMap(ENCHANTMENT, LEVEL_CODEC)
```

Verifiziert anhand des Klassen-Bytecodes von
`net/minecraft/world/item/enchantment/ItemEnchantments.class` aus
`minecraft-merged-deobf-26.2.jar`. Die Klasse führt genau ein Feld:
`enchantments`. Ein `levels`-Wrapper existiert nicht.

## Korrigierte Fassung

```mcfunction
clear @p
item replace entity @p weapon.mainhand with minecraft:netherite_sword[minecraft:enchantments={sharpness:5,unbreaking:3,mending:1}]
item replace entity @p armor.head with minecraft:netherite_helmet[minecraft:enchantments={protection:4,unbreaking:3,mending:1}]
item replace entity @p armor.chest with minecraft:netherite_chestplate[minecraft:enchantments={protection:4,unbreaking:3,mending:1}]
item replace entity @p armor.legs with minecraft:netherite_leggings[minecraft:enchantments={protection:4,unbreaking:3,mending:1}]
item replace entity @p armor.feet with minecraft:netherite_boots[minecraft:enchantments={protection:4,unbreaking:3,mending:1,feather_falling:4}]
item replace entity @p weapon.offhand with minecraft:totem_of_undying
give @p minecraft:end_crystal 128
give @p minecraft:obsidian 128
give @p minecraft:glowstone 128
give @p minecraft:respawn_anchor 128
give @p minecraft:cobweb 64
give @p minecraft:enchanted_golden_apple 64
give @p minecraft:totem_of_undying 25
```

## Unterschiede zur alten Fassung

| | alt | neu |
|---|---|---|
| Struktur | `{levels:{"minecraft:sharpness":5}}` | `{sharpness:5}` |
| Namensraum | `minecraft:` bei jedem Schlüssel | weggelassen |

Der Namensraum darf bei Enchantment-Schlüsseln entfallen; `minecraft` ist der
Vorgabe-Namensraum. Das ist der Grund, warum die alte Fassung doppelt fehlschlug:
sowohl der Wrapper als auch die Präfixe passten nicht zum Parser.

## Ablageort

```
<Welt>/datapacks/provipvp_kit/data/provipvp/function/kit.mcfunction
```

## Teststatus

Die Syntax ist aus dem Bytecode der 26.2-Komponente abgeleitet und die Datei liegt
korrekt in beiden Welten. **Nicht im Spiel verifiziert** — beim nächsten
Weltladen sollte im Log keine Zeile `Failed to load function provipvp:kit`
mehr erscheinen. `/function provipvp:kit` sollte das Set austeilen.
