/**
 * Safe Presser — non-destructive example
 * - Removes destructive commands (no mass delete/ban/kick/spam)
 * - Adds owner-only guard, permission checks, validation
 * - Uses discord.js v14 style
 */

const { Client, GatewayIntentBits, Partials, EmbedBuilder, PermissionsBitField, ActivityType } = require("discord.js");
const { cyan, greenBright, red } = require("chalk");
const config = require("./config/config.json"); // { token, prefix, ownerID }
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.GuildEmojisAndStickers
  ],
  partials: [Partials.Message, Partials.Channel, Partials.Reaction]
});

const PREFIX = config.prefix || "!";
const OWNER_ID = config.ownerID;

// Utility: owner-only guard
function isOwner(userId) {
  return userId === OWNER_ID;
}

// Simple embed helper
function makeEmbed(title, desc) {
  return new EmbedBuilder()
    .setTitle(title)
    .setDescription(desc)
    .setColor(0x2f3136)
    .setTimestamp();
}

client.once("ready", () => {
  console.clear();
  console.log(cyan(`Safe Presser Bot ready — Logged in as ${client.user.tag}`));
  client.user.setActivity("helping admins", { type: ActivityType.Playing });
});

client.on("messageCreate", async (message) => {
  try {
    if (message.author.bot) return;
    if (!message.guild) return; // only operate in guilds

    const content = message.content.trim();
    if (!content.startsWith(PREFIX)) return;

    const [cmd, ...rawArgs] = content.slice(PREFIX.length).split(/\s+/);
    const args = rawArgs.join(" ").trim();

    // Help
    if (cmd === "help") {
      const embed = makeEmbed("Safe Presser — Help", `
**Commands (safe)**
\`${PREFIX}listchannels\` — Lists channels in this guild.
\`${PREFIX}createchannels <amount> <name>\` — Create up to 5 channels (owner-only).
\`${PREFIX}listroles\` — Lists roles in this guild.
\`${PREFIX}createrole <name>\` — Create one role (owner-only).
`);
      return message.channel.send({ embeds: [embed] });
    }

    // List channels (safe, read-only)
    if (cmd === "listchannels") {
      const chs = message.guild.channels.cache
        .map((c) => `${c.name} (${c.id}) — ${c.type}`)
        .slice(0, 50); // limit output
      return message.channel.send({ embeds: [makeEmbed("Channels (first 50)", chs.length ? chs.join("\n") : "No channels")] });
    }

    // Create channels — **owner-only** and limited amount
    if (cmd === "createchannels") {
      if (!isOwner(message.author.id)) return message.reply("Only the bot owner can use this command.");
      if (!message.guild.members.me.permissions.has(PermissionsBitField.Flags.ManageChannels)) {
        return message.reply("I need Manage Channels permission to create channels.");
      }

      const parts = args.split(/\s+/);
      const amount = parseInt(parts[0], 10);
      const name = parts.slice(1).join(" ") || `${message.author.username}-channel`;

      if (isNaN(amount) || amount <= 0) return message.reply("Usage: createchannels <amount> <name>");
      if (amount > 5) return message.reply("Limit: you can create at most 5 channels with this safe command.");

      const created = [];
      for (let i = 0; i < amount; i++) {
        // create text channels only and wait for resolution to avoid spamming the API
        const ch = await message.guild.channels.create({
          name: `${name}-${i + 1}`,
          type: 0 // GuildText in v14 numeric constant; alternatively use ChannelType.GuildText
        }).catch(err => {
          console.error(red("Channel create error:"), err);
        });
        if (ch) created.push(ch.name);
      }
      return message.reply(`Created channels: ${created.join(", ")}`);
    }

    // List roles (safe)
    if (cmd === "listroles") {
      const roles = message.guild.roles.cache
        .map(r => `${r.name} (${r.id})`)
        .slice(0, 50);
      return message.channel.send({ embeds: [makeEmbed("Roles (first 50)", roles.length ? roles.join("\n") : "No roles")] });
    }

    // Create a single role — owner-only, safe defaults
    if (cmd === "createrole") {
      if (!isOwner(message.author.id)) return message.reply("Only the bot owner can use this command.");
      if (!message.guild.members.me.permissions.has(PermissionsBitField.Flags.ManageRoles)) {
        return message.reply("I need Manage Roles permission to create roles.");
      }
      const roleName = args || `role-${Date.now().toString().slice(-4)}`;
      const role = await message.guild.roles.create({
        name: roleName,
        reason: `Created by ${message.author.tag} via safe bot command`
      }).catch(err => {
        console.error(red("Role create error:"), err);
        return null;
      });
      if (role) return message.reply(`Created role: ${role.name}`);
      return message.reply("Failed to create role (check logs).");
    }

    // Unknown
    return message.reply(`Unknown command. Try \`${PREFIX}help\``);
  } catch (err) {
    console.error(red("Message handler error:"), err);
    message.channel.send({ embeds: [makeEmbed("Error", "An internal error occurred — check bot logs.")] }).catch(() => {});
  }
});

process.on("unhandledRejection", (err) => {
  console.error(red("Unhandled promise rejection:"), err);
});

client.login(process.env.TOKEN || config.token).catch(err => {
  console.error(red("Login failed:"), err);
});
