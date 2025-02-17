const {
    ChannelType,
    PermissionsBitField,
    ModalBuilder,
    TextInputBuilder,
    TextInputStyle,
    ActionRowBuilder,
    ButtonBuilder,
    ButtonStyle,
    UserSelectMenuBuilder,
} = require('discord.js');
const { QuickDB } = require('quick.db');
const client = require('../../index');
const fs = require('fs');
const path = require('path');
const db = new QuickDB();

// Add delay utility
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));

module.exports = {
    name: "pvc"
};

// Voice State Update Handler
client.on("voiceStateUpdate", async (oldState, newState) => {
    const guild = newState.guild || oldState.guild;
    const configPath = path.join(__dirname, '../../data', guild.id, 'pvc.json');
    
    if (!fs.existsSync(configPath)) return;
    const config = JSON.parse(fs.readFileSync(configPath));

    // Channel creation logic
    if (newState.channelId === config.generatorId) {
        let tempChannel;
        try {
            tempChannel = await guild.channels.create({
                name: `${newState.member.user.displayName}'s VC`,
                type: ChannelType.GuildVoice,
                parent: config.categoryId,
            });
            await db.set(`Temporary_${tempChannel.id}_${newState.member.user.id}`, tempChannel.id);

            // Wait 1 second and verify user is still connected
            await delay(1000);
            const refreshedMember = await guild.members.fetch(newState.member.id);
            
            if (refreshedMember.voice.channelId === config.generatorId) {
                await refreshedMember.voice.setChannel(tempChannel);
            } else {
                await tempChannel.delete().catch(console.error);
                await db.delete(`Temporary_${tempChannel.id}_${newState.member.user.id}`);
            }

        } catch (error) {
            console.error("PVC Creation Error:", error);
            // Cleanup if channel was created
            if (tempChannel) {
                await tempChannel.delete().catch(console.error);
                await db.delete(`Temporary_${tempChannel.id}_${newState.member.user.id}`);
            }
        }
    }

    // Ownership transfer and cleanup logic
    if (oldState.channelId && !newState.channelId) {
        try {
            const wasOwner = await db.get(`Temporary_${oldState.channelId}_${oldState.member.user.id}`);
            if (!wasOwner) return;

            const channel = guild.channels.cache.get(oldState.channelId);
            if (!channel) return;

            const members = channel.members.filter(m => !m.user.bot);
            
            if (members.size > 0) {
                const newOwner = members.first();
                await db.delete(`Temporary_${oldState.channelId}_${oldState.member.user.id}`);
                await db.set(`Temporary_${oldState.channelId}_${newOwner.id}`, oldState.channelId);
                
                try {
                    await channel.setName(`${newOwner.user.displayName}'s VC`);
                } catch (error) {
                    console.error("Channel Rename Error:", error);
                }
            } else {
                await channel.delete();
                await db.delete(`Temporary_${oldState.channelId}_${oldState.member.user.id}`);
            }
        } catch (error) {
            console.error("PVC Ownership Transfer Error:", error);
        }
    }
});

// Interaction Create Handler
client.on("interactionCreate", async (interaction) => {
    // Ignore bots
    if (interaction.user.bot) return;

    // Handle user select menus
    if (interaction.isUserSelectMenu() && interaction.customId.startsWith('UsersManagerSelect')) {
        const [_, guildId] = interaction.customId.split('-');
        if (!guildId) return;

        try {
            await interaction.deferUpdate();
            const voiceChannel = interaction.member.voice.channel;
            if (!voiceChannel) return;

            // Verify ownership
            const tempChannelId = await db.get(`Temporary_${voiceChannel.id}_${interaction.user.id}`);
            if (tempChannelId !== voiceChannel.id) return;

            const selectedUserId = interaction.values[0];
            // Store the selected user for later button action
            await db.set(`UsersManagerSelection_${interaction.user.id}_${guildId}`, selectedUserId);

            await interaction.followUp({
                content: `✅ Selected <@${selectedUserId}> for management. Now use the action buttons below.`,
                ephemeral: true
            });

        } catch (error) {
            console.error('User Selection Error:', error);
            await interaction.followUp({
                content: '❌ Failed to select user!',
                ephemeral: true
            });
        }
        return;
    }

    // Handle buttons and modals
    if (!interaction.isButton() && !interaction.isModalSubmit()) return;

    const [action, guildId] = interaction.customId?.split('-') || [];
    if (!guildId) return;
    
    const configPath = path.join(__dirname, '../../data', guildId, 'pvc.json');
    if (!fs.existsSync(configPath)) return;

    // Load control message ID
    const pvcMessagePath = path.join(__dirname, '../../data', guildId, 'pvc_message.json');
    if (!fs.existsSync(pvcMessagePath)) return;
    const { messageId: controlMessageId } = JSON.parse(fs.readFileSync(pvcMessagePath));

    // Verify button interaction is from the control message
    if (interaction.isButton() && interaction.message.id !== controlMessageId) return;

    // For modals, defer update only if not already deferred
    if (interaction.isModalSubmit() && !interaction.deferred && !interaction.replied) {
        await interaction.deferUpdate({ ephemeral: true });
    }

    const voiceChannel = interaction.member.voice.channel;
    if (!voiceChannel) {
        if (interaction.isModalSubmit()) {
            await interaction.followUp({ 
                content: '❌ You must be in a voice channel!', 
                ephemeral: true 
            });
        } else {
            await interaction.reply({ 
                content: '❌ You must be in a voice channel!', 
                ephemeral: true 
            });
        }
        return;
    }

    try {
        // Check ownership for non-UsersManager actions
        if (!interaction.customId.startsWith('UsersManager')) {
            const tempChannelId = await db.get(`Temporary_${voiceChannel.id}_${interaction.user.id}`);
            if (tempChannelId !== voiceChannel.id) {
                return interaction.reply({ 
                    content: '❌ You are not the owner of this channel!', 
                    ephemeral: true 
                });
            }
        }

        if (interaction.isButton()) {
            // Handle modal interactions first
            if (action === 'RenameChannel' || action === 'Customize_UserLimit') {
                let modal;
                if (action === 'RenameChannel') {
                    modal = new ModalBuilder()
                        .setCustomId(`RenameModal-${guildId}`)
                        .setTitle('Rename Channel');
                    const nameInput = new TextInputBuilder()
                        .setCustomId('name')
                        .setLabel('New Channel Name')
                        .setStyle(TextInputStyle.Short)
                        .setMaxLength(50);
                    modal.addComponents(new ActionRowBuilder().addComponents(nameInput));
                } else {
                    modal = new ModalBuilder()
                        .setCustomId(`Customize_UsersLimit-${guildId}`)
                        .setTitle('Set User Limit');
                    const limitInput = new TextInputBuilder()
                        .setCustomId('limit')
                        .setLabel('Max Users (0-99)')
                        .setStyle(TextInputStyle.Short)
                        .setMaxLength(2)
                        .setPlaceholder('0-99');
                    modal.addComponents(new ActionRowBuilder().addComponents(limitInput));
                }
                return await interaction.showModal(modal);
            }

            // If button interaction is for UsersManager actions
            if (interaction.customId.startsWith('UsersManager_')) {
                await interaction.deferUpdate();
                // Retrieve the stored selected user id
                const selectedUserId = await db.get(`UsersManagerSelection_${interaction.user.id}_${guildId}`);
                if (!selectedUserId) {
                    return interaction.followUp({
                        content: '❌ No user selected! Please select a user first from the menu.',
                        ephemeral: true
                    });
                }
                
                // Check if selected user is in the voice channel
                const targetMember = await voiceChannel.guild.members.fetch(selectedUserId).catch(() => null);
                if (!targetMember || !voiceChannel.members.has(targetMember.id)) {
                    return interaction.followUp({
                        content: '❌ Selected user is not in this voice channel!',
                        ephemeral: true
                    });
                }

                switch(action) {
                    case 'UsersManager_Mute':
                        await targetMember.voice.setMute(true);
                        break;
                    case 'UsersManager_Unmute':
                        await targetMember.voice.setMute(false);
                        break;
                    case 'UsersManager_Deafen':
                        await targetMember.voice.setDeaf(true);
                        break;
                    case 'UsersManager_Undeafen':
                        await targetMember.voice.setDeaf(false);
                        break;
                    default:
                        break;
                }

                const actionWord = action.split('_')[1].toLowerCase();
                await interaction.followUp({
                    content: `✅ Successfully ${actionWord}ed ${targetMember.user.tag}!`,
                    ephemeral: true
                });
                return;
            }

            // Defer update for other buttons if not already done
            if (!interaction.deferred && !interaction.replied) {
                await interaction.deferUpdate();
            }
            
            switch(action) {
                case 'LockChannel':
                    await voiceChannel.permissionOverwrites.set([
                        { id: interaction.guild.roles.everyone.id, deny: [PermissionsBitField.Flags.Connect] },
                        { id: interaction.user.id, allow: [PermissionsBitField.Flags.Connect] }
                    ]);
                    break;

                case 'UnlockChannel':
                    await voiceChannel.permissionOverwrites.edit(interaction.guild.id, { Connect: null });
                    break;

                case 'HideChannel':
                    await voiceChannel.permissionOverwrites.set([
                        { id: interaction.guild.roles.everyone.id, deny: [PermissionsBitField.Flags.ViewChannel] },
                        { id: interaction.user.id, allow: [PermissionsBitField.Flags.ViewChannel] }
                    ]);
                    break;

                case 'UnhideChannel':
                    await voiceChannel.permissionOverwrites.edit(interaction.guild.id, { ViewChannel: null });
                    break;

                case 'Mute':
                    voiceChannel.members.forEach(m => {
                        if (m.id !== interaction.user.id && !m.user.bot) m.voice.setMute(true);
                    });
                    break;

                case 'Unmute':
                    voiceChannel.members.forEach(m => {
                        if (m.id !== interaction.user.id && !m.user.bot) m.voice.setMute(false);
                    });
                    break;

                case 'Disconnect':
                    voiceChannel.members.forEach(m => {
                        if (m.id !== interaction.user.id && !m.user.bot) m.voice.disconnect();
                    });
                    break;

                case 'Delete_Channel':
                    await db.delete(`Temporary_${voiceChannel.id}_${interaction.user.id}`);
                    await voiceChannel.delete();
                    break;

                case 'UsersManager':
                    const userSelectRow = new ActionRowBuilder().addComponents(
                        new UserSelectMenuBuilder()
                            .setCustomId(`UsersManagerSelect-${guildId}`)
                            .setPlaceholder('Select a user to manage')
                    );

                    const actionRow = new ActionRowBuilder().addComponents(
                        new ButtonBuilder()
                            .setCustomId(`UsersManager_Mute-${guildId}`)
                            .setLabel('Mute')
                            .setStyle(ButtonStyle.Danger),
                        new ButtonBuilder()
                            .setCustomId(`UsersManager_Unmute-${guildId}`)
                            .setLabel('Unmute')
                            .setStyle(ButtonStyle.Success),
                        new ButtonBuilder()
                            .setCustomId(`UsersManager_Deafen-${guildId}`)
                            .setLabel('Deafen')
                            .setStyle(ButtonStyle.Danger),
                        new ButtonBuilder()
                            .setCustomId(`UsersManager_Undeafen-${guildId}`)
                            .setLabel('Undeafen')
                            .setStyle(ButtonStyle.Success)
                    );

                    await interaction.followUp({
                        content: 'Select a user and then click an action button:',
                        components: [userSelectRow, actionRow],
                        ephemeral: true
                    });
                    break;
            }
        }

        // Handle modals
        if (interaction.isModalSubmit()) {
            // Removed redundant deferUpdate() call to avoid "InteractionAlreadyReplied" error.
            if (interaction.customId.startsWith('RenameModal')) {
                const newName = interaction.fields.getTextInputValue('name');
                try {
                    await voiceChannel.setName(newName);
                    await interaction.followUp({
                        content: `✅ Channel renamed to: ${newName}`,
                        ephemeral: true
                    });
                } catch (error) {
                    console.error('Rename Error:', error);
                    await interaction.followUp({
                        content: '❌ Failed to rename channel!',
                        ephemeral: true
                    });
                }
            } else if (interaction.customId.startsWith('Customize_UsersLimit')) {
                const newLimit = interaction.fields.getTextInputValue('limit');
            
                // Ensure input is a number
                if (!/^\d+$/.test(newLimit)) {
                    return interaction.followUp({
                        content: '❌ Invalid input! Please enter a number between 0 and 99.',
                        ephemeral: true
                    });
                }
            
                const parsedLimit = parseInt(newLimit);
            
                // Ensure number is within valid range
                if (isNaN(parsedLimit) || parsedLimit < 0 || parsedLimit > 99) {
                    return interaction.followUp({
                        content: '❌ Invalid user limit! Please enter a number between 0 and 99.',
                        ephemeral: true
                    });
                }
            
                try {
                    // Set limit to unlimited if user enters 0
                    await voiceChannel.setUserLimit(parsedLimit === 0 ? 0 : parsedLimit);
                    
                    await interaction.followUp({
                        content: `✅ User limit set to: ${parsedLimit === 0 ? 'Unlimited' : parsedLimit}`,
                        ephemeral: true
                    });
                } catch (error) {
                    console.error('User Limit Error:', error);
                    await interaction.followUp({
                        content: '❌ Failed to set user limit!',
                        ephemeral: true
                    });
                }
            }            
        }

    } catch (error) {
        console.error('PVC Interaction Error:', error);
        if (interaction.isModalSubmit() || interaction.isButton()) {
            if (interaction.replied || interaction.deferred) {
                await interaction.followUp({ 
                    content: '❌ Action failed!', 
                    ephemeral: true 
                });
            } else {
                await interaction.reply({ 
                    content: '❌ Action failed!', 
                    ephemeral: true 
                });
            }
        }
    }
});
