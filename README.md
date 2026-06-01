import discord
from discord.ext import commands
import asyncio

# Setup intents
intents = discord.Intents.default()
intents.message_content = True
intents.guilds = True
intents.bans = True

# Initialize bot with a prefix
bot = commands.Bot(command_prefix="!", intents=intents)

@bot.event
async def on_ready():
    print(f'🤖 Bot is online as {bot.user.name}')
    print('------')

### 1. MASS UNBAN ###
@bot.command(name="unbanall")
@commands.has_permissions(ban_members=True)
async def unban_all(ctx):
    """Lifts all bans in the server."""
    await ctx.send("🔄 Fetching ban list... This might take a moment.")
    
    ban_entries = []
    async for ban in ctx.guild.bans():
        ban_entries.append(ban.user)
        
    if not ban_entries:
        return await ctx.send("✅ No banned users found.")

    await ctx.send(f"🔨 Unbanning {len(ban_entries)} users...")
    
    count = 0
    for user in ban_entries:
        try:
            await ctx.guild.unban(user)
            count += 1
            # Tiny delay to prevent heavy rate limiting
            await asyncio.sleep(0.5) 
        except discord.HTTPException:
            continue

    await ctx.send(f"✨ Successfully unbanned {count}/{len(ban_entries)} users.")

### 2. MASS REMOVE CHANNELS ###
@bot.command(name="cleanorder")
@commands.has_permissions(manage_channels=True)
async def clean_channels(ctx):
    """Deletes all existing channels and creates one fresh text channel."""
    await ctx.send("⚠️ WARNING: Deleting all channels in 5 seconds. Type `cancel` to stop.")

    def check(m):
        return m.author == ctx.author and m.content.lower() == 'cancel' and m.channel == ctx.channel

    try:
        # Wait for a cancel message
        await bot.wait_for('message', check=check, timeout=5.0)
        return await ctx.send("❌ Operation cancelled.")
    except asyncio.TimeoutError:
        pass

    # Create a backup channel first so the bot has a place to exist while deleting others
    fallback_channel = await ctx.guild.create_text_channel(name="server-reset")
    await fallback_channel.send("🧹 Cleaning channels... Please wait.")

    for channel in ctx.guild.channels:
        if channel == fallback_channel:
            continue
        try:
            await channel.delete()
            await asyncio.sleep(0.3)
        except discord.HTTPException:
            continue

    await fallback_channel.send("✨ All old channels have been purged.")

### 3. MASS CLEAN WEBHOOKS ###
@bot.command(name="cleanwebhooks")
@commands.has_permissions(manage_webhooks=True)
async def clean_webhooks(ctx):
    """Deletes every webhook in the server."""
    await ctx.send("🔄 Fetching and deleting all webhooks...")
    
    try:
        webhooks = await ctx.guild.webhooks()
    except discord.HTTPException:
        return await ctx.send("❌ Failed to fetch webhooks.")

    if not webhooks:
        return await ctx.send("✅ No webhooks found.")

    count = 0
    for webhook in webhooks:
        try:
            await webhook.delete()
            count += 1
            await asyncio.sleep(0.2)
        except discord.HTTPException:
            continue

    await ctx.send(f"✨ Successfully deleted {count} webhooks.")

# Error handling for missing permissions
@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("❌ You don't have the required permissions to run this command.")
    else:
        print(f"Error: {error}")

### 4. MASS REMOVE SPAM ROLES ###
@bot.command(name="cleanroles")
@commands.has_permissions(manage_roles=True)
async def clean_roles(ctx):
    """Deletes all roles below the bot's highest role, except @everyone and managed roles."""
    await ctx.send("🔄 Fetching roles... This will skip bot integration roles.")
    
    # Filter out @everyone, managed roles (bot roles), and roles higher than the bot
    roles_to_delete = [
        role for role in ctx.guild.roles 
        if not role.is_default() 
        and not role.managed 
        and role < ctx.guild.me.top_role
    ]
    
    if not roles_to_delete:
        return await ctx.send("✅ No deletable roles found.")
        
    await ctx.send(f"🛡️ Deleting {len(roles_to_delete)} roles...")
    
    count = 0
    for role in roles_to_delete:
        try:
            await role.delete()
            count += 1
            await asyncio.sleep(0.3)
        except discord.HTTPException:
            continue
            
    await ctx.send(f"✨ Successfully deleted {count}/{len(roles_to_delete)} roles.")

### 5. MASS KICK/PRUNE NEW MEMBERS (ANTI-RAID) ###
@bot.command(name="prune")
@commands.has_permissions(kick_members=True)
async def prune_members(ctx, days: int = 1):
    """Prunes members who haven't logged in for X days. Perfect for removing raid accounts."""
    if days < 1 or days > 30:
        return await ctx.send("❌ Days must be between 1 and 30.")
        
    await ctx.send(f"🔄 Checking how many members haven't been active in {days} day(s)...")
    
    try:
        # Check how many would be kicked before actually doing it
        estimate = await ctx.guild.estimate_pruned_members(days=days)
        if estimate == 0:
            return await ctx.send("✅ No inactive members found to prune.")
            
        await ctx.send(f"⚠️ This will kick approximately **{estimate} members**. Type `confirm` within 10 seconds to proceed.")
        
        def check(m):
            return m.author == ctx.author and m.content.lower() == 'confirm' and m.channel == ctx.channel
            
        await bot.wait_for('message', check=check, timeout=10.0)
        
        # Execute prune
        kicked = await ctx.guild.prune_members(days=days, reason="Server cleanup after raid.")
        await ctx.send(f"✨ Successfully pruned {kicked} inactive/raid members.")
        
    except asyncio.TimeoutError:
        await ctx.send("❌ Prune cancelled (timeout).")
    except discord.HTTPException as e:
        await ctx.send(f"❌ Failed to prune: {e}")

### 6. RESET SERVER IDENTITY ###
@bot.command(name="resetserver")
@commands.has_permissions(manage_guild=True)
async def reset_server(ctx, *, new_name: str = "Recovered Server"):
    """Resets the server name, removes any offensive nuker banner/icon."""
    await ctx.send("🔄 Resetting server identity...")
    
    try:
        # Edit server details, removing icon and banner by setting them to None
        await ctx.guild.edit(
            name=new_name,
            icon=None,
            banner=None,
            splash=None,
            description="This server is currently under reconstruction."
        )
        await ctx.send(f"✨ Server name reset to **{new_name}**. Icon and banners removed.")
    except discord.HTTPException as e:
        await ctx.send(f"❌ Failed to reset server settings: {e}")
