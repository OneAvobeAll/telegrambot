import re
import logging
from telegram import Update
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    filters,
    ContextTypes
)

# Enter bot token here
BOT_TOKEN = "8045453724:AAGw3MbWBhWduZ0RP2dVyx095QMWR_013mI"

# Set up logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# State keys
REPLACEMENT_USERNAME = "replacement_username"

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Send welcome message and instructions"""
    await update.message.reply_text(
        "✨ Welcome to Username Replacer Bot! ✨\n\n"
        "Here's how to use me:\n"
        "1. Set your replacement username with /setusername @your_username\n"
        "2. Forward any media (photos, videos, audio, files) or text to me\n\n"
        "I'll replace all @usernames in the content with your specified username!\n\n"
        "You can change your replacement username at any time.",
        parse_mode="HTML"
    )

async def set_username(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Set the replacement username"""
    if not context.args:
        await update.message.reply_text(
            "❌ Please specify a username!\n"
            "Example: /setusername @your_username"
        )
        return
    
    input_text = " ".join(context.args)
    username_match = re.search(r'@(\w+)', input_text)
    
    if not username_match:
        await update.message.reply_text(
            "❌ Invalid username format!\n"
            "Please use @ followed by letters/numbers (e.g., @your_username)"
        )
        return
    
    replacement_username = username_match.group(0)
    context.user_data[REPLACEMENT_USERNAME] = replacement_username
    
    await update.message.reply_text(
        f"✅ Replacement username set to {replacement_username}!\n\n"
        "Now forward any media or text to me to process it."
    )

async def handle_forwarded_content(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Process all forwarded content and replace usernames"""
    message = update.message
    
    # Check replacement username
    replacement_username = context.user_data.get(REPLACEMENT_USERNAME)
    if not replacement_username:
        await message.reply_text(
            "⚠️ You haven't set a replacement username yet!\n"
            "Please set one with /setusername @your_username"
        )
        return

    # Process text messages
    if message.text:
        original_text = message.text
        new_text = re.sub(r'@\w+', replacement_username, original_text)
        await message.reply_text(new_text)
        return

    # Process media content
    media_handlers = {
        'photo': {
            'obj': message.photo,
            'method': context.bot.send_photo,
            'args': {'photo': None},
            'get_obj': lambda: message.photo[-1] if message.photo else None,
            'attrs': ['duration', 'width', 'height']
        },
        'video': {
            'obj': message.video,
            'method': context.bot.send_video,
            'args': {'video': None},
            'get_obj': lambda: message.video,
            'attrs': ['duration', 'width', 'height', 'supports_streaming']
        },
        'audio': {
            'obj': message.audio,
            'method': context.bot.send_audio,
            'args': {'audio': None},
            'get_obj': lambda: message.audio,
            'attrs': ['duration', 'title', 'performer']
        },
        'document': {
            'obj': message.document,
            'method': context.bot.send_document,
            'args': {'document': None},
            'get_obj': lambda: message.document,
            'attrs': ['filename']  # Special handling below
        },
        'voice': {
            'obj': message.voice,
            'method': context.bot.send_voice,
            'args': {'voice': None},
            'get_obj': lambda: message.voice,
            'attrs': ['duration']
        },
        'sticker': {
            'obj': message.sticker,
            'method': context.bot.send_sticker,
            'args': {'sticker': None},
            'get_obj': lambda: message.sticker,
            'attrs': []
        },
        'animation': {
            'obj': message.animation,
            'method': context.bot.send_animation,
            'args': {'animation': None},
            'get_obj': lambda: message.animation,
            'attrs': ['duration', 'width', 'height']
        }
    }

    for media_type, handler in media_handlers.items():
        media_obj = handler['obj']
        if media_obj:
            try:
                # Get the media object
                media_file = handler['get_obj']()
                
                # Prepare arguments
                kwargs = handler['args'].copy()
                kwargs[list(kwargs.keys())[0]] = media_file.file_id
                
                # Process caption if exists
                caption = message.caption or ""
                if caption:
                    kwargs['caption'] = re.sub(r'@\w+', replacement_username, caption)
                
                # Add optional metadata
                for attr in handler['attrs']:
                    if hasattr(media_file, attr):
                        # Special case for document filename
                        if media_type == 'document' and attr == 'filename':
                            kwargs['filename'] = getattr(media_file, 'file_name', 'file')
                        else:
                            kwargs[attr] = getattr(media_file, attr)
                
                # Send media
                await handler['method'](
                    chat_id=message.chat_id,
                    **kwargs
                )
                logger.info(f"Processed {media_type} with username replacement")
            except Exception as e:
                logger.error(f"Error processing {media_type}: {str(e)}")
                await message.reply_text(f"❌ Failed to process {media_type}. Please try again.")
            return
    
    # Unsupported content type
    await message.reply_text("❌ Unsupported content type! Please forward media files or text.")

async def show_username(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Show current replacement username"""
    replacement_username = context.user_data.get(REPLACEMENT_USERNAME)
    
    if replacement_username:
        await update.message.reply_text(
            f"Your current replacement username is: {replacement_username}\n"
            "To change it, use /setusername @new_username"
        )
    else:
        await update.message.reply_text(
            "You haven't set a replacement username yet!\n"
            "Set one with /setusername @your_username"
        )

def main() -> None:
    """Start the bot."""
    application = Application.builder().token(BOT_TOKEN).build()

    # Register handlers
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("setusername", set_username))
    application.add_handler(CommandHandler("myusername", show_username))
    
    # Handle all forwarded content
    application.add_handler(MessageHandler(
        filters.FORWARDED & (
            filters.TEXT |
            filters.PHOTO |
            filters.VIDEO |
            filters.AUDIO |
            filters.Document.ALL |
            filters.VOICE |
            filters.Sticker.ALL |
            filters.ANIMATION
        ), 
        handle_forwarded_content
    ))

    # Start the bot
    application.run_polling()
    logger.info("Bot is now running...")

if __name__ == '__main__':
    main()
