
# Discord Bot with n8n Integration: Complete Setup Guide

## Part 1: n8n Webhook Setup

1. **Create a New Workflow in n8n**:
   - Log into your n8n instance
   - Create a new workflow (click "+" or "Create Workflow")
   - Name it (e.g., "Discord Bot Handler")

2. **Add a Webhook Trigger**:
   - Add a new node (click the "+" button)
   - Search for and select "Webhook"
   - Configure it:
     - Authentication: None (or set up as needed)
     - HTTP Method: POST
     - Path: Leave default or customize (e.g., "/discord-bot")
     - Click "Execute Node" to initialize it

3. **Get Webhook URL**:
   - After executing the node, you'll see the Production Webhook URL
   - Copy this URL - you'll need it for your Discord bot's .env file
   - It looks like: `https://your-n8n-instance.com/webhook/path...`

4. **Add Processing Nodes**:
   - Connect nodes for your processing logic
   - For example, an AI service integration or database lookup

5. **Format Response Correctly**:
   - Add a "Function" node before the final response
   - Use code like:
   ```javascript
   // Process your data and format the response
   return {
     json: {
       answer: "Your response here"
     }
   };
   ```

6. **Add Respond to Webhook Node**:
   - Add a "Respond to Webhook" node as the final step
   - Configure it to return the data from your Function node
   - Ensure it includes the "answer" field that your bot will look for

7. **Activate the Workflow**:
   - Toggle the "Active" switch in the top-right corner
   - Save your workflow

## Part 2: Discord Developer Portal Setup

1. **Create a Discord Application**:
   - Go to [Discord Developer Portal](https://discord.com/developers/applications)
   - Click "New Application" and give it a name
   - Navigate to the "Bot" tab
   - Click "Add Bot"
  
    

2. **Configure Bot Permissions**:
   - Under the "Bot" tab, enable these intents:
     - Message Content Intent (CRITICAL)
   - In "Bot Permissions", select:
     - Read Messages/View Channels
     - Send Messages
     - Read Message History

![image](https://github.com/user-attachments/assets/b7387bb9-dfa8-488e-850b-7b01a10240ed)


3. **Get Bot Token**:
   - Under the "Bot" tab, click "Reset Token" or "Copy" if you already have one
   - Keep this token secure - it's how your code authenticates as your bot

4. **Generate OAuth2 URL**:
   - Go to "OAuth2" → "URL Generator"
   - Select scopes: "bot" and "applications.commands"
   - Select bot permissions: "Read Messages/View Channels", "Send Messages", "Read Message History"
   - Copy the generated URL, paste it in the browser and open it to invite your bot to your server
![image](https://github.com/user-attachments/assets/4c584bbd-e684-47a1-aa7a-15ae8e488026)
![image](https://github.com/user-attachments/assets/fb33a612-ab40-4f65-b74c-ad32a723a702)
![image](https://github.com/user-attachments/assets/06ef0ed9-32ee-46d0-9144-0254c213a525)


## Part 3: Server Setup

1. **Install Node.js (v20.x)**:
   ```bash
   # Remove old Node.js versions
   sudo apt purge nodejs npm
   sudo apt autoremove
   
   # Add Node.js v20.x repository
   curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
   
   # Install Node.js
   sudo apt install -y nodejs
   
   # Verify installation
   node -v  # Should show v20.x.x
   npm -v   # Should show 10.x.x
   ```

2. **Create Project Directory**:
   ```bash
   # Create and navigate to project directory
   mkdir discord-bot
   cd discord-bot
   ```

3. **Initialize Project & Install Dependencies**:
   ```bash
   # Initialize npm project
   npm init -y
   
   # Install required packages
   npm install discord.js axios dotenv
   ```

## Part 4: Bot Configuration

1. **Create Environment File**:
   ```bash
   # Create .env file
   nano .env
   ```

2. **Add Environment Variables**:
   ```
   DISCORD_BOT_TOKEN=your_bot_token_from_developer_portal
   N8N_WEBHOOK_URL=your_n8n_webhook_production_url_from_part1
   # Optional: Uncomment to restrict bot to a specific channel
   # TARGET_CHANNEL_ID=your_channel_id
   ```

3. **Create Bot Code**:
   ```bash
   # Create index.js file
   nano index.js
   ```

4. **Add Bot Code to index.js**:

// IMPORTANT: Discord has a 2000 character message limit. To handle AI's long responses, we implement message chunking logic below. This splits large responses at paragraph or sentence boundaries and sends them as multiple sequential messages.

```javascript
require('dotenv').config();
const { Client, GatewayIntentBits } = require('discord.js');
const axios = require('axios');

const token = process.env.DISCORD_BOT_TOKEN;
const n8nWebhookUrl = process.env.N8N_WEBHOOK_URL;
const targetChannelId = process.env.TARGET_CHANNEL_ID;

// Create client with required intents
const client = new Client({
    intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent // IMPORTANT: Must be enabled in Developer Portal!
    ]
});

// When bot is ready
client.once('ready', () => {
    console.log(`Logged in as ${client.user.tag}!`);
    if (targetChannelId) {
      console.log(`Listening for mentions in specific channel: ${targetChannelId}`);
    } else {
      console.log(`Listening for mentions in any channel.`);
    }
});

// When a message is created
client.on('messageCreate', async message => {
    // 1. Ignore messages from self OR when bot is NOT mentioned
    if (message.author.bot || !message.mentions.has(client.user)) {
       return;
    }

    // 2. Optional: Only respond in target channel
    if (targetChannelId && message.channel.id !== targetChannelId) {
        console.log(`Ignoring mention in wrong channel: ${message.channel.id}`);
        return;
    }

    // 3. Extract the question (remove the mention)
    const question = message.content.replace(`<@${client.user.id}>`, '').trim();
    const channelId = message.channel.id;

    // 3.5 Check if a question was actually asked
    if (!question) {
        console.log(`Bot mentioned without a question by ${message.author.tag}. Ignoring.`);
        // Optional: Send a help message
        // message.reply("Mention me and ask a question, e.g. `@MyBot How are you?`");
        return;
    }

    console.log(`Received question from channel ${channelId} by ${message.author.tag}: "${question}"`);

    // Show "typing..." continuously while waiting for response
    const typingInterval = setInterval(() => {
        message.channel.sendTyping().catch(err => {
            console.error('Error sending typing indicator:', err);
            clearInterval(typingInterval);
        });
    }, 5000); // Send typing indicator every 5 seconds

    // For very long processes, send a status update
    let statusMessageSent = false;
    const statusTimeout = setTimeout(() => {
        message.reply("I'm still working on your request. This might take a bit longer than usual.");
        statusMessageSent = true;
    }, 20000); // Send status message after 20 seconds

    try {
        // 4. Send the question and channel ID to n8n with increased timeout
        const responseFromN8n = await axios.post(n8nWebhookUrl, {
            question: question,
            channelId: channelId,
            userId: message.author.id,
            userName: message.author.username
        }, {
            timeout: 300000 // Increase timeout to 5 minutes (300 seconds)
        });

        // Clear the intervals
        clearInterval(typingInterval);
        clearTimeout(statusTimeout);

        // Debug logging of complete response
        console.log('Complete response from n8n:', JSON.stringify(responseFromN8n.data));

        // 5. Enhanced response processing with multiple format handling
        let answer;
        
        // Try different response structures
        if (responseFromN8n.data && responseFromN8n.data.answer) {
            // Standard format: { "answer": "..." }
            answer = responseFromN8n.data.answer;
        } else if (Array.isArray(responseFromN8n.data) && responseFromN8n.data[0]?.answer) {
            // Array format: [{ "answer": "..." }]
            answer = responseFromN8n.data[0].answer;
        } else if (typeof responseFromN8n.data === 'string') {
            // Direct string format
            answer = responseFromN8n.data;
        } else {
            // Fallback: Try to find any structure
            console.log('No known response structure found, trying fallback parsing...');
            const jsonStr = JSON.stringify(responseFromN8n.data);
            console.log('Raw data:', jsonStr);
            
            if (jsonStr.includes('"answer"')) {
                try {
                    // Manual extraction of answer field
                    const match = jsonStr.match(/"answer"\s*:\s*"([^"]+)"/);
                    if (match && match[1]) {
                        answer = match[1];
                    }
                } catch (parseError) {
                    console.error('Error with manual parsing:', parseError);
                }
            }
        }

        // 6. Send the answer to the Discord channel
        if (answer) {
            // Check if the answer exceeds Discord's character limit (2000)
            if (answer.length <= 2000) {
                // Standard reply if message is short enough
                message.reply(answer);
                console.log(`Sent answer from n8n: "${answer.substring(0, 100)}..."`);
            } else {
                // Split long messages into chunks of 1900 characters (leaving room for formatting)
                const chunks = [];
                let temp = answer;
                
                while (temp.length > 0) {
                    // Find a good breaking point (preferably at a paragraph or sentence)
                    let breakPoint = 1900;
                    if (temp.length > breakPoint) {
                        // Try to find paragraph break
                        const paragraphBreak = temp.lastIndexOf('\n\n', breakPoint);
                        if (paragraphBreak > breakPoint / 2) {
                            breakPoint = paragraphBreak;
                        } else {
                            // Try to find sentence break
                            const sentenceBreak = temp.lastIndexOf('. ', breakPoint);
                            if (sentenceBreak > breakPoint / 2) {
                                breakPoint = sentenceBreak + 1; // Include the period
                            }
                        }
                    } else {
                        breakPoint = temp.length;
                    }
                    
                    chunks.push(temp.substring(0, breakPoint));
                    temp = temp.substring(breakPoint);
                }
                
                // Send first chunk as a reply to the original message
                await message.reply(chunks[0]);
                console.log(`Sent first chunk of answer (${chunks[0].length} chars)`);
                
                // Send remaining chunks as follow-up messages
                for (let i = 1; i < chunks.length; i++) {
                    await message.channel.send(chunks[i]);
                    console.log(`Sent chunk ${i+1} of ${chunks.length} (${chunks[i].length} chars)`);
                    
                    // Small delay between messages to avoid rate limiting
                    if (i < chunks.length - 1) {
                        await new Promise(resolve => setTimeout(resolve, 1000));
                    }
                }
            }
        } else {
            message.reply("I did not receive a valid answer from my n8n workflow.");
            console.log('Received empty or invalid answer structure from n8n:', responseFromN8n.data);
        }

    } catch (error) {
        // Clear the intervals on error
        clearInterval(typingInterval);
        clearTimeout(statusTimeout);

        console.error('Error interacting with n8n or Discord:', error.message);
        message.reply("Oops, something went wrong when communicating with n8n. Please try again later.");
        // Detailed error handling for n8n response errors
        if (error.response) {
          console.error('n8n responded with status:', error.response.status);
          console.error('n8n response data:', error.response.data);
        } else if (error.request) {
          console.error('No response received from n8n request:', error.request);
        } else {
          console.error('Error setting up request:', error.message);
        }
    }
});

// Log in the bot
client.login(token);
```

## Part 5: Running the Bot

1. **Test Run**:
   ```bash
   # Start bot
   node index.js
   ```

2. **Persistent Deployment with PM2**:
   ```bash
   # Install PM2 globally
   npm install -g pm2
   
   # Start bot with PM2
   pm2 start index.js --name discord-bot
   
   # Check status
   pm2 status
   
   # View logs
   pm2 logs discord-bot
   
   # Set up PM2 to start on server boot
   pm2 startup
   
   # Run the command shown by PM2 startup
   # It will look like: sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u your_username --hp /home/your_username
   
   # Save the current PM2 configuration
   pm2 save
   ```

## Part 6: Testing and Troubleshooting

1. **Test in Discord**:
   - On your Discord server, mention the bot: `@YourBot What's the weather like?`
   - Your bot should respond with the answer from n8n

2. **Common Issues and Solutions**:
   - **Bot not responding**: Check PM2 logs for errors with `pm2 logs discord-bot`
   - **"Message Content Intent" errors**: Make sure you enabled it in Discord Developer Portal
   - **n8n connection issues**: Verify your webhook URL is correct and the workflow is active
   - **Format errors**: Ensure your n8n workflow returns a response with an `answer` field

3. **Viewing Raw Response Data**:
   - Check the logs with `pm2 logs discord-bot`
   - Look for the line "Complete response from n8n:" to see the exact format being received

This revised sequence ensures you have everything ready in the correct order, making the setup process smoother and preventing potential configuration issues.
