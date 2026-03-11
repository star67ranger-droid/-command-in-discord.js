# ⚡ Guide Complet — Slash Commands & Components V2 Discord.js v14

> Tout ce qu'il faut savoir pour créer des slash commands modernes avec des réponses riches grâce aux Components V2.

---

## Table des matières

- [C'est quoi une slash command ?](#cest-quoi-une-slash-command-)
- [Structure du projet](#structure-du-projet)
- [Enregistrer les commandes](#enregistrer-les-commandes)
- [Anatomie d'une commande](#anatomie-dune-commande)
- [Les types d'options](#les-types-doptions)
- [Choices — options à choix fixe](#choices--options-à-choix-fixe)
- [Sous-commandes](#sous-commandes)
- [Autocomplete](#autocomplete)
- [Répondre correctement](#répondre-correctement)
- [Permissions](#permissions)
- [Handler d'interactions](#handler-dinteractions)
- [Réponses avec Components V2](#réponses-avec-components-v2)
- [Exemple complet — commande /profil](#exemple-complet--commande-profil)
- [Règles d'or](#règles-dor)

---

## C'est quoi une slash command ?

Une slash command est une commande déclenchée par `/` dans Discord. Elle est enregistrée directement auprès de l'API Discord, ce qui lui donne un menu autocomplete natif et des options typées. C'est le standard moderne — oublie les prefix commands.

---

## Structure du projet

```
my-bot/
├── commands/
│   ├── ping.js
│   └── profil.js
├── events/
│   ├── ready.js
│   └── interactionCreate.js
├── deploy-commands.js
├── index.js
└── config.json
```

**`config.json`**

```json
{
  "token": "TON_BOT_TOKEN",
  "clientId": "TON_CLIENT_ID",
  "guildId": "TON_GUILD_ID"
}
```

**`index.js`**

```js
const {
  Client,
  GatewayIntentBits,
  Collection,
} = require('discord.js');
const fs = require('fs');
const path = require('path');
const { token } = require('./config.json');

const client = new Client({ intents: [GatewayIntentBits.Guilds] });
client.commands = new Collection();

// Chargement des commandes
const commandsPath = path.join(__dirname, 'commands');
for (const file of fs.readdirSync(commandsPath).filter(f => f.endsWith('.js'))) {
  const cmd = require(path.join(commandsPath, file));
  if (cmd.data && cmd.execute) client.commands.set(cmd.data.name, cmd);
}

// Chargement des events
const eventsPath = path.join(__dirname, 'events');
for (const file of fs.readdirSync(eventsPath).filter(f => f.endsWith('.js'))) {
  const event = require(path.join(eventsPath, file));
  client[event.once ? 'once' : 'on'](event.name, (...args) => event.execute(...args, client));
}

client.login(token);
```

---

## Enregistrer les commandes

Les commandes doivent être envoyées à l'API Discord via `deploy-commands.js`. Tu lances ce script **manuellement** à chaque fois que tu modifies ou ajoutes une commande.

```js
const { REST, Routes } = require('discord.js');
const { token, clientId, guildId } = require('./config.json');
const fs = require('fs');
const path = require('path');

const commands = [];
const commandsPath = path.join(__dirname, 'commands');

for (const file of fs.readdirSync(commandsPath).filter(f => f.endsWith('.js'))) {
  const { data } = require(path.join(commandsPath, file));
  if (data) commands.push(data.toJSON());
}

const rest = new REST({ version: '10' }).setToken(token);

(async () => {
  // Guild = instantané (dev) | Global = jusqu'à 1h de délai (prod)
  await rest.put(Routes.applicationGuildCommands(clientId, guildId), { body: commands });
  console.log('✅ Commandes enregistrées');
})();
```

> **Règle importante :** en développement utilise toujours les guild commands (`applicationGuildCommands`), elles sont instantanées. Les global commands (`applicationCommands`) peuvent prendre jusqu'à 1 heure à se propager.

---

## Anatomie d'une commande

Chaque commande est un fichier JS qui exporte deux choses : `data` (la définition) et `execute` (la logique).

```js
const { SlashCommandBuilder } = require('discord.js');

module.exports = {
  data: new SlashCommandBuilder()
    .setName('ping')                          // minuscules, pas d'espaces, max 32 caractères
    .setDescription('Répond avec la latence du bot'), // max 100 caractères

  async execute(interaction, client) {
    await interaction.deferReply();

    const latency = Date.now() - interaction.createdTimestamp;
    const apiPing = client.ws.ping;

    await interaction.editReply(`🏓 Pong ! Latence : **${latency}ms** | API : **${apiPing}ms**`);
  }
};
```

---

## Les types d'options

Tu ajoutes des arguments à ta commande avec les méthodes `add...Option()`. Chaque option peut être `.setRequired(true)` ou optionnelle.

```js
const { SlashCommandBuilder, ChannelType } = require('discord.js');

new SlashCommandBuilder()
  .setName('demo')
  .setDescription('Exemple')
  .addStringOption(opt => opt
    .setName('texte')
    .setDescription('Un texte')
    .setRequired(true)
    .setMinLength(1)
    .setMaxLength(100)
  )
  .addIntegerOption(opt => opt
    .setName('nombre')
    .setDescription('Un entier')
    .setMinValue(1)
    .setMaxValue(100)
  )
  .addNumberOption(opt => opt
    .setName('decimal')
    .setDescription('Un nombre décimal')
  )
  .addBooleanOption(opt => opt
    .setName('actif')
    .setDescription('Vrai ou faux')
  )
  .addUserOption(opt => opt
    .setName('cible')
    .setDescription('Un utilisateur')
  )
  .addChannelOption(opt => opt
    .setName('salon')
    .setDescription('Un salon textuel uniquement')
    .addChannelTypes(ChannelType.GuildText)
  )
  .addRoleOption(opt => opt
    .setName('role')
    .setDescription('Un rôle')
  )
  .addAttachmentOption(opt => opt
    .setName('fichier')
    .setDescription('Un fichier uploadé')
  )
```

Pour les récupérer dans `execute` :

```js
const texte   = interaction.options.getString('texte');
const nombre  = interaction.options.getInteger('nombre');
const user    = interaction.options.getUser('cible');
const salon   = interaction.options.getChannel('salon');
const role    = interaction.options.getRole('role');
const fichier = interaction.options.getAttachment('fichier');
```

> Si une option est optionnelle et non renseignée, tu récupères `null`. Utilise `?? 'valeur par défaut'` pour gérer ça proprement.

---

## Choices — options à choix fixe

Si tu veux limiter les valeurs possibles, utilise `.addChoices()`. Discord affichera un menu déroulant.

```js
.addStringOption(opt => opt
  .setName('couleur')
  .setDescription('Choisir une couleur')
  .setRequired(true)
  .addChoices(
    { name: 'Rouge', value: 'red' },
    { name: 'Bleu',  value: 'blue' },
    { name: 'Vert',  value: 'green' },
  )
)
```

Le `name` c'est ce que l'utilisateur voit. La `value` c'est ce que ton code reçoit.

---

## Sous-commandes

Quand tu as plusieurs actions liées, utilise les sous-commandes plutôt que de créer plusieurs commandes séparées.

```js
new SlashCommandBuilder()
  .setName('config')
  .setDescription('Configuration du bot')

  // Sous-commande simple
  .addSubcommand(sub => sub
    .setName('voir')
    .setDescription('Voir la config actuelle')
  )

  // Groupe de sous-commandes
  .addSubcommandGroup(group => group
    .setName('logs')
    .setDescription('Gérer les logs')
    .addSubcommand(sub => sub
      .setName('set')
      .setDescription('Définir le salon de logs')
      .addChannelOption(opt => opt.setName('salon').setDescription('Salon cible').setRequired(true))
    )
    .addSubcommand(sub => sub
      .setName('reset')
      .setDescription('Réinitialiser les logs')
    )
  )
```

Dans `execute`, tu récupères la sous-commande utilisée :

```js
const group = interaction.options.getSubcommandGroup(false); // false = pas obligatoire
const sub   = interaction.options.getSubcommand();

if (!group && sub === 'voir') {
  return interaction.reply('Config actuelle : ...');
}

if (group === 'logs' && sub === 'set') {
  const salon = interaction.options.getChannel('salon');
  return interaction.reply(`Salon de logs défini : ${salon}`);
}

if (group === 'logs' && sub === 'reset') {
  return interaction.reply('Logs réinitialisés.');
}
```

---

## Autocomplete

L'autocomplete affiche des suggestions dynamiques pendant que l'utilisateur tape. Contrairement aux choices, les suggestions peuvent venir d'une base de données ou changer en temps réel.

**Dans `data` :**
```js
.addStringOption(opt => opt
  .setName('langue')
  .setDescription('Langage de programmation')
  .setAutocomplete(true)
)
```

**Dans ton fichier de commande, ajoute la méthode `autocomplete` :**
```js
async autocomplete(interaction) {
  const input   = interaction.options.getFocused().toLowerCase();
  const langues = ['JavaScript', 'TypeScript', 'Python', 'Rust', 'Go', 'Java'];
  const filtres = langues
    .filter(l => l.toLowerCase().includes(input))
    .slice(0, 25); // max 25 suggestions

  await interaction.respond(filtres.map(l => ({ name: l, value: l })));
},
```

**Dans ton handler `interactionCreate` :**
```js
if (interaction.isAutocomplete()) {
  const cmd = client.commands.get(interaction.commandName);
  if (cmd?.autocomplete) await cmd.autocomplete(interaction);
}
```

---

## Répondre correctement

Discord te donne **3 secondes** pour répondre à une interaction. Si tu dépasses ce délai, elle expire et tout plante.

| Méthode | Quand l'utiliser |
|---|---|
| `interaction.reply()` | Réponse immédiate, logique instantanée |
| `interaction.deferReply()` | Opération async (API, base de données...) |
| `interaction.editReply()` | Après un `deferReply()` |
| `interaction.followUp()` | Message supplémentaire après une reply |
| `interaction.deleteReply()` | Supprimer la réponse originale |

**Pattern recommandé pour toute commande async :**

```js
async execute(interaction) {
  await interaction.deferReply(); // ← toujours en premier

  try {
    const data = await fetch('https://api.exemple.com/data');
    const json = await data.json();
    await interaction.editReply(`Résultat : ${json.value}`);
  } catch (err) {
    await interaction.editReply({ content: '❌ Une erreur est survenue.', ephemeral: true });
  }
}
```

**Ephemeral** — rend la réponse visible uniquement par l'utilisateur. Indispensable pour les erreurs.

```js
await interaction.reply({ content: 'Seulement toi vois ça.', ephemeral: true });

// ou pour un defer éphémère :
await interaction.deferReply({ ephemeral: true });
```

---

## Permissions

```js
const { PermissionFlagsBits } = require('discord.js');

new SlashCommandBuilder()
  .setName('ban')
  .setDescription('Bannir un membre')
  .setDefaultMemberPermissions(PermissionFlagsBits.BanMembers) // cache la commande si pas la perm
  .setDMPermission(false)                                       // désactiver en DM
```

> **Attention :** `setDefaultMemberPermissions` est une indication visuelle pour Discord, pas une vraie sécurité. Il faut **toujours** vérifier les permissions dans `execute` également.

```js
async execute(interaction) {
  if (!interaction.member.permissions.has(PermissionFlagsBits.BanMembers)) {
    return interaction.reply({ content: '❌ Tu n\'as pas la permission.', ephemeral: true });
  }

  const membre = interaction.options.getMember('membre');

  // Vérifier la hiérarchie des rôles
  if (membre.roles.highest.position >= interaction.member.roles.highest.position) {
    return interaction.reply({ content: '❌ Tu ne peux pas bannir ce membre.', ephemeral: true });
  }

  await membre.ban();
  await interaction.reply(`✅ **${membre.user.tag}** a été banni.`);
}
```

---

## Handler d'interactions

Tout passe par l'event `interactionCreate`. C'est là que tu dispatches vers la bonne commande.

```js
module.exports = {
  name: 'interactionCreate',

  async execute(interaction, client) {

    // Slash commands
    if (interaction.isChatInputCommand()) {
      const cmd = client.commands.get(interaction.commandName);
      if (!cmd) return;

      try {
        await cmd.execute(interaction, client);
      } catch (err) {
        console.error(err);
        const msg = { content: '❌ Erreur lors de l\'exécution.', ephemeral: true };
        if (interaction.replied || interaction.deferred) await interaction.followUp(msg);
        else await interaction.reply(msg);
      }
    }

    // Autocomplete
    if (interaction.isAutocomplete()) {
      const cmd = client.commands.get(interaction.commandName);
      if (cmd?.autocomplete) await cmd.autocomplete(interaction);
    }

    // Boutons
    if (interaction.isButton()) {
      // gérer tes boutons ici via interaction.customId
    }

    // Select menus
    if (interaction.isAnySelectMenu()) {
      // gérer tes select menus ici via interaction.customId
    }

  }
};
```

> Le `try/catch` est indispensable en production. Sans ça, une erreur dans une commande laisse l'interaction en suspens côté utilisateur.

---

## Réponses avec Components V2

Components V2 remplace les embeds classiques. Il permet de créer des messages riches avec du texte, des images, des boutons et des menus dans un bloc visuel cohérent.

La règle absolue : **toujours ajouter `flags: MessageFlags.IsComponentsV2`** dans ta réponse, sinon Discord ignore tout.

### Les composants disponibles

**TextDisplay** — texte avec Markdown complet :

```js
new TextDisplayBuilder().setContent('## Titre\nDescription en **gras**.')
```

**Separator** — ligne de séparation :

```js
new SeparatorBuilder().setDivider(true).setSpacing(SeparatorSpacingSize.Small)
```

**Section** — texte + accessoire (thumbnail ou bouton). L'accessoire est obligatoire :

```js
new SectionBuilder()
  .addTextDisplayComponents(
    new TextDisplayBuilder().setContent('## Titre'),
    new TextDisplayBuilder().setContent('Description.')
  )
  .setThumbnailAccessory(
    new ThumbnailBuilder({ media: { url: 'https://example.com/image.png' } })
  )
```

**MediaGallery** — une ou plusieurs images en carrousel :

```js
new MediaGalleryBuilder().addItems(
  new MediaGalleryItemBuilder().setURL('https://example.com/image1.png').setDescription('Légende'),
  new MediaGalleryItemBuilder().setURL('https://example.com/image2.png')
)
```

**File** — pièce jointe référencée dans le container :

```js
const attachment = new AttachmentBuilder('./fichier.json').setName('fichier.json');
const fileComponent = new FileBuilder().setURL('attachment://fichier.json');
// Ne pas oublier de passer files: [attachment] dans la réponse
```

**Container** — regroupe tous les composants ci-dessus :

```js
const container = new ContainerBuilder()
  .setAccentColor(0x5865F2)
  .addTextDisplayComponents(new TextDisplayBuilder().setContent('Contenu'))
  .addSeparatorComponents(new SeparatorBuilder().setDivider(true).setSpacing(SeparatorSpacingSize.Small))
  .addSectionComponents(/* ... */)
  .addMediaGalleryComponents(/* ... */)
  .addFileComponents(/* ... */);
```

### Ce qui va dans le container vs en dehors

| Dans le container `add...Components()` | En dehors dans `components: []` |
|---|---|
| `TextDisplayBuilder` | `ActionRowBuilder` (boutons classiques, select menus) |
| `SeparatorBuilder` | |
| `SectionBuilder` | |
| `MediaGalleryBuilder` | |
| `FileBuilder` | |

### Envoyer un message Components V2

```js
await interaction.reply({
  flags: MessageFlags.IsComponentsV2,
  components: [container, selectRow] // selectRow en dehors du container
});
```

---

## Exemple complet — commande /profil

```js
const {
  SlashCommandBuilder,
  MessageFlags,
  ContainerBuilder,
  TextDisplayBuilder,
  SeparatorBuilder,
  SeparatorSpacingSize,
  SectionBuilder,
  ThumbnailBuilder,
  ButtonBuilder,
  ButtonStyle,
  MediaGalleryBuilder,
  MediaGalleryItemBuilder,
  ActionRowBuilder,
  ChannelSelectMenuBuilder
} = require('discord.js');

module.exports = {
  data: new SlashCommandBuilder()
    .setName('profil')
    .setDescription("Afficher le profil d'un utilisateur")
    .addUserOption(opt => opt
      .setName('user')
      .setDescription('Utilisateur cible (toi par défaut)')
    ),

  async execute(interaction) {
    const user       = interaction.options.getUser('user') ?? interaction.user;
    const member     = interaction.guild.members.cache.get(user.id);
    const avatar     = user.displayAvatarURL({ extension: 'png', size: 256 });
    const joinDate   = member?.joinedAt?.toLocaleDateString('fr-FR') ?? 'Inconnu';
    const createDate = user.createdAt.toLocaleDateString('fr-FR');
    const roles      = member?.roles.cache
      .filter(r => r.id !== interaction.guild.id)
      .map(r => `<@&${r.id}>`)
      .join(' ') || 'Aucun rôle';

    const container = new ContainerBuilder()
      .setAccentColor(0x5865F2)

      // En-tête avec avatar
      .addSectionComponents(
        new SectionBuilder()
          .addTextDisplayComponents(
            new TextDisplayBuilder().setContent(`## 👤 ${user.username}`),
            new TextDisplayBuilder().setContent(`ID : \`${user.id}\``)
          )
          .setThumbnailAccessory(
            new ThumbnailBuilder({ media: { url: avatar } })
          )
      )

      .addSeparatorComponents(
        new SeparatorBuilder().setDivider(true).setSpacing(SeparatorSpacingSize.Small)
      )

      // Informations
      .addTextDisplayComponents(
        new TextDisplayBuilder().setContent('### 📋 Informations'),
        new TextDisplayBuilder().setContent(`📅 **Compte créé le** : ${createDate}`),
        new TextDisplayBuilder().setContent(`📥 **A rejoint le** : ${joinDate}`),
        new TextDisplayBuilder().setContent(`🏷️ **Rôles** : ${roles}`)
      )

      .addSeparatorComponents(
        new SeparatorBuilder().setDivider(true).setSpacing(SeparatorSpacingSize.Small)
      )

      // Galerie avatar
      .addMediaGalleryComponents(
        new MediaGalleryBuilder().addItems(
          new MediaGalleryItemBuilder().setURL(avatar).setDescription('Avatar')
        )
      )

      .addSeparatorComponents(
        new SeparatorBuilder().setDivider(true).setSpacing(SeparatorSpacingSize.Small)
      )

      // Lien vers l'avatar
      .addSectionComponents(
        new SectionBuilder()
          .addTextDisplayComponents(
            new TextDisplayBuilder().setContent('🔗 **Voir l\'avatar en plein écran**')
          )
          .setButtonAccessory(
            new ButtonBuilder()
              .setLabel('Ouvrir')
              .setURL(avatar)
              .setStyle(ButtonStyle.Link)
          )
      );

    // Select menu en dehors du container (obligatoire)
    const selectRow = new ActionRowBuilder().addComponents(
      new ChannelSelectMenuBuilder()
        .setCustomId('select_channel')
        .setPlaceholder('Envoyer le profil dans un salon...')
    );

    await interaction.reply({
      flags: MessageFlags.IsComponentsV2,
      components: [container, selectRow]
    });
  }
};
```

---

## Règles d'or

- 🕐 Toujours `deferReply()` dès qu'il y a une opération async, même légère.
- 🔒 Toujours vérifier les permissions côté code, pas seulement via Discord.
- 🛠️ Guild commands en dev, global commands uniquement en production.
- 🔄 Relancer `deploy-commands.js` après chaque modification de structure.
- 🔕 Répondre en `ephemeral` pour les erreurs pour ne pas polluer le salon.
- 🛡️ Mettre un `try/catch` dans le handler sur chaque `execute()`.
- ⚠️ Toujours ajouter `MessageFlags.IsComponentsV2` sinon les composants V2 ne s'affichent pas.
- 📋 Ne jamais mettre un `ActionRowBuilder` dans un `ContainerBuilder` — il va toujours en dehors.
- 🖼️ Une `SectionBuilder` doit toujours avoir un accessoire (thumbnail ou bouton), sinon erreur.

---

## Ressources

- [Discord.js Guide](https://discordjs.guide/)
- [Discord.js Docs](https://discord.js.org/)
- [Discord Developer Docs — Application Commands](https://discord.com/developers/docs/interactions/application-commands)
- [Discord Developer Docs — Components Overview](https://discord.com/developers/docs/components/overview)
- [Component Types Reference](https://discord.com/developers/docs/components/reference)
