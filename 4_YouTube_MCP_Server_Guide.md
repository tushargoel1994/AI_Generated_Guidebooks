# Building a YouTube MCP Server with FastMCP

> **A Complete Step-by-Step Guide to Create, Deploy, and Use a YouTube MCP Server**

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Setup Google Cloud Project](#setup-google-cloud-project)
4. [Project Setup](#project-setup)
5. [Building the YouTube MCP Server](#building-the-youtube-mcp-server)
6. [Testing the Server](#testing-the-server)
7. [Integrating with Claude Desktop](#integrating-with-claude-desktop)
8. [Usage Examples](#usage-examples)
9. [Troubleshooting](#troubleshooting)

---

## Overview

This guide walks you through building a YouTube MCP (Model Context Protocol) server using FastMCP that provides three key functionalities:

1. **Search Videos** - Search for YouTube videos using the `search.list` API
2. **Get Channel Information** - Retrieve channel details using the `channels.list` API
3. **Manage Subscriptions** - List, add, and remove channel subscriptions using `subscriptions.list`, `subscriptions.insert`, and `subscriptions.delete` APIs

### What You'll Build

A production-ready MCP server that:
- ✅ Authenticates with YouTube using OAuth 2.0
- ✅ Exposes 5 MCP tools for YouTube operations
- ✅ Handles errors gracefully
- ✅ Works seamlessly with Claude Desktop
- ✅ Supports both read and write operations

---

## Prerequisites

### Required Software

1. **Python 3.10 or higher**
   ```bash
   python --version
   # Should show Python 3.10+
   ```

2. **uv (Python Package Manager)**
   ```bash
   # Install uv (recommended)
   curl -LsSf https://astral.sh/uv/install.sh | sh
   
   # Or using pip
   pip install uv
   ```

3. **Claude Desktop** (for testing)
   - Download from: https://claude.ai/download

4. **Google Account** with access to Google Cloud Console

### Required Knowledge

- Basic Python programming
- Understanding of APIs and OAuth 2.0
- Command line basics

---

## Setup Google Cloud Project

### Step 1: Create Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)

2. Click **"Select a project"** dropdown at the top → **"New Project"**

3. Enter project details:
   - **Project name:** `youtube-mcp-server`
   - **Organization:** (leave default or select your org)
   - Click **"Create"**

4. Wait for project creation (takes ~30 seconds)

### Step 2: Enable YouTube Data API v3

1. In the Google Cloud Console, ensure your new project is selected

2. Navigate to **"APIs & Services"** → **"Library"**

3. Search for **"YouTube Data API v3"**

4. Click on **"YouTube Data API v3"**

5. Click **"Enable"** button

6. Wait for API to be enabled

### Step 3: Configure OAuth Consent Screen

1. Navigate to **"APIs & Services"** → **"OAuth consent screen"**

2. Select **User Type**:
   - Choose **"External"** (for personal use)
   - Click **"Create"**

3. Fill in **App Information**:
   - **App name:** `YouTube MCP Server`
   - **User support email:** Your email address
   - **Developer contact information:** Your email address
   - Click **"Save and Continue"**

4. **Scopes** screen:
   - Click **"Add or Remove Scopes"**
   - Search and select these scopes:
     - `https://www.googleapis.com/auth/youtube.readonly`
     - `https://www.googleapis.com/auth/youtube.force-ssl`
   - Click **"Update"**
   - Click **"Save and Continue"**

5. **Test Users** screen:
   - Click **"+ Add Users"**
   - Add your Google account email
   - Click **"Add"**
   - Click **"Save and Continue"**

6. **Summary** screen:
   - Review settings
   - Click **"Back to Dashboard"**

### Step 4: Create OAuth 2.0 Credentials

1. Navigate to **"APIs & Services"** → **"Credentials"**

2. Click **"+ Create Credentials"** → **"OAuth client ID"**

3. Configure OAuth client:
   - **Application type:** `Desktop app`
   - **Name:** `YouTube MCP Desktop Client`
   - Click **"Create"**

4. **Download credentials:**
   - A popup appears showing your client ID and secret
   - Click **"Download JSON"**
   - Save the file as `client_secret.json`
   - Click **"OK"**

5. **Important:** Keep `client_secret.json` secure - it contains sensitive credentials

---

## Project Setup

### Step 1: Create Project Directory

```bash
# Create project directory
mkdir youtube-mcp-server
cd youtube-mcp-server

# Create project structure
mkdir -p src/youtube_mcp
touch src/youtube_mcp/__init__.py
```

### Step 2: Initialize Python Project

```bash
# Initialize with uv
uv init

# Create virtual environment
uv venv

# Activate virtual environment
# On macOS/Linux:
source .venv/bin/activate

# On Windows:
.venv\Scripts\activate
```

### Step 3: Install Dependencies

```bash
# Install required packages
uv add fastmcp
uv add google-auth-oauthlib
uv add google-auth-httplib2
uv add google-api-python-client
uv add python-dotenv
```

### Step 4: Setup Project Files

```bash
# Create main server file
touch src/youtube_mcp/server.py

# Create environment file
touch .env

# Create gitignore
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
.venv/
venv/
env/

# Credentials
client_secret.json
token.pickle
.env

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
EOF
```

### Step 5: Move Credentials File

```bash
# Move your downloaded client_secret.json to project root
# If it's in Downloads:
mv ~/Downloads/client_secret.json ./client_secret.json

# Verify it's there
ls -la client_secret.json
```

---

## Building the YouTube MCP Server

### Step 1: Create the Main Server File

Create `src/youtube_mcp/server.py` with the following code:

```python
"""
YouTube MCP Server
A FastMCP server providing YouTube search, channel info, and subscription management.
"""

import os
import pickle
from typing import Optional, List, Dict, Any
from pathlib import Path

from fastmcp import FastMCP
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

# Initialize FastMCP server
mcp = FastMCP("YouTube MCP Server")

# YouTube API configuration
SCOPES = [
    'https://www.googleapis.com/auth/youtube.readonly',
    'https://www.googleapis.com/auth/youtube.force-ssl'
]

# Paths
PROJECT_ROOT = Path(__file__).parent.parent.parent
CLIENT_SECRET_FILE = PROJECT_ROOT / 'client_secret.json'
TOKEN_FILE = PROJECT_ROOT / 'token.pickle'


def get_authenticated_service():
    """
    Authenticate with YouTube API using OAuth 2.0.
    
    Returns:
        YouTube API service object
    """
    creds = None
    
    # Load existing credentials
    if TOKEN_FILE.exists():
        with open(TOKEN_FILE, 'rb') as token:
            creds = pickle.load(token)
    
    # If credentials are invalid or don't exist, authenticate
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            if not CLIENT_SECRET_FILE.exists():
                raise FileNotFoundError(
                    f"client_secret.json not found at {CLIENT_SECRET_FILE}. "
                    "Please download it from Google Cloud Console."
                )
            
            flow = InstalledAppFlow.from_client_secrets_file(
                str(CLIENT_SECRET_FILE), 
                SCOPES
            )
            creds = flow.run_local_server(port=0)
        
        # Save credentials for future use
        with open(TOKEN_FILE, 'wb') as token:
            pickle.dump(creds, token)
    
    return build('youtube', 'v3', credentials=creds)


@mcp.tool()
def search_videos(
    query: str,
    max_results: int = 10,
    order: str = "relevance",
    video_type: str = "any"
) -> str:
    """
    Search for YouTube videos.
    
    Args:
        query: Search query string (e.g., "python tutorial")
        max_results: Number of results to return (1-50, default: 10)
        order: Sort order - "relevance", "date", "rating", "viewCount", "title" (default: "relevance")
        video_type: Type filter - "any", "episode", "movie" (default: "any")
    
    Returns:
        JSON string with video search results
    """
    try:
        # Validate parameters
        if not query or not query.strip():
            return "Error: Query cannot be empty"
        
        if not 1 <= max_results <= 50:
            return "Error: max_results must be between 1 and 50"
        
        valid_orders = ["relevance", "date", "rating", "viewCount", "title"]
        if order not in valid_orders:
            return f"Error: order must be one of {valid_orders}"
        
        valid_types = ["any", "episode", "movie"]
        if video_type not in valid_types:
            return f"Error: video_type must be one of {valid_types}"
        
        # Get authenticated service
        youtube = get_authenticated_service()
        
        # Execute search
        search_params = {
            'part': 'snippet',
            'q': query,
            'maxResults': max_results,
            'order': order,
            'type': 'video'
        }
        
        if video_type != "any":
            search_params['videoType'] = video_type
        
        request = youtube.search().list(**search_params)
        response = request.execute()
        
        # Format results
        results = []
        for item in response.get('items', []):
            video_id = item['id']['videoId']
            snippet = item['snippet']
            
            results.append({
                'video_id': video_id,
                'title': snippet['title'],
                'description': snippet['description'],
                'channel_title': snippet['channelTitle'],
                'channel_id': snippet['channelId'],
                'published_at': snippet['publishedAt'],
                'thumbnail': snippet['thumbnails']['default']['url'],
                'video_url': f"https://www.youtube.com/watch?v={video_id}"
            })
        
        if not results:
            return f"No videos found for query: {query}"
        
        # Format output
        output = f"Found {len(results)} videos for '{query}':\n\n"
        for i, video in enumerate(results, 1):
            output += f"{i}. {video['title']}\n"
            output += f"   Channel: {video['channel_title']}\n"
            output += f"   URL: {video['video_url']}\n"
            output += f"   Published: {video['published_at']}\n"
            output += f"   Description: {video['description'][:150]}...\n\n"
        
        return output
    
    except HttpError as e:
        return f"YouTube API Error: {e.resp.status} - {e.content.decode()}"
    except Exception as e:
        return f"Error: {str(e)}"


@mcp.tool()
def get_channel_info(
    channel_id: Optional[str] = None,
    channel_username: Optional[str] = None,
    include_statistics: bool = True
) -> str:
    """
    Get information about a YouTube channel.
    
    Args:
        channel_id: YouTube channel ID (e.g., "UC_x5XG1OV2P6uZZ5FSM9Ttw")
        channel_username: Channel username (e.g., "@GoogleDevelopers")
        include_statistics: Include subscriber count, view count, etc. (default: True)
    
    Returns:
        JSON string with channel information
    
    Note:
        Provide either channel_id OR channel_username, not both.
    """
    try:
        # Validate parameters
        if not channel_id and not channel_username:
            return "Error: Must provide either channel_id or channel_username"
        
        if channel_id and channel_username:
            return "Error: Provide only channel_id OR channel_username, not both"
        
        # Get authenticated service
        youtube = get_authenticated_service()
        
        # Build request parameters
        parts = ['snippet', 'contentDetails']
        if include_statistics:
            parts.append('statistics')
        
        request_params = {
            'part': ','.join(parts)
        }
        
        if channel_id:
            request_params['id'] = channel_id
        else:
            # Remove @ if present
            username = channel_username.lstrip('@')
            request_params['forUsername'] = username
        
        # Execute request
        request = youtube.channels().list(**request_params)
        response = request.execute()
        
        # Check if channel exists
        if not response.get('items'):
            identifier = channel_id or channel_username
            return f"Error: No channel found with identifier: {identifier}"
        
        # Get first (and should be only) result
        channel = response['items'][0]
        
        # Extract data
        snippet = channel['snippet']
        content_details = channel['contentDetails']
        
        output = f"Channel Information:\n\n"
        output += f"Title: {snippet['title']}\n"
        output += f"Channel ID: {channel['id']}\n"
        output += f"Description: {snippet['description'][:300]}...\n"
        output += f"Custom URL: {snippet.get('customUrl', 'N/A')}\n"
        output += f"Published At: {snippet['publishedAt']}\n"
        output += f"Country: {snippet.get('country', 'Not specified')}\n"
        output += f"Uploads Playlist: {content_details['relatedPlaylists']['uploads']}\n"
        
        if include_statistics and 'statistics' in channel:
            stats = channel['statistics']
            output += f"\nStatistics:\n"
            output += f"  Subscribers: {stats.get('subscriberCount', 'Hidden')}\n"
            output += f"  Total Views: {stats.get('viewCount', '0')}\n"
            output += f"  Total Videos: {stats.get('videoCount', '0')}\n"
        
        output += f"\nThumbnail: {snippet['thumbnails']['default']['url']}\n"
        output += f"Channel URL: https://www.youtube.com/channel/{channel['id']}\n"
        
        return output
    
    except HttpError as e:
        return f"YouTube API Error: {e.resp.status} - {e.content.decode()}"
    except Exception as e:
        return f"Error: {str(e)}"


@mcp.tool()
def list_subscriptions(
    max_results: int = 25,
    order: str = "relevance"
) -> str:
    """
    List the authenticated user's YouTube channel subscriptions.
    
    Args:
        max_results: Number of subscriptions to return (1-50, default: 25)
        order: Sort order - "relevance", "unread", "alphabetical" (default: "relevance")
    
    Returns:
        JSON string with list of subscriptions
    """
    try:
        # Validate parameters
        if not 1 <= max_results <= 50:
            return "Error: max_results must be between 1 and 50"
        
        valid_orders = ["relevance", "unread", "alphabetical"]
        if order not in valid_orders:
            return f"Error: order must be one of {valid_orders}"
        
        # Get authenticated service
        youtube = get_authenticated_service()
        
        # Execute request
        request = youtube.subscriptions().list(
            part='snippet,contentDetails',
            mine=True,
            maxResults=max_results,
            order=order
        )
        response = request.execute()
        
        # Check if user has subscriptions
        if not response.get('items'):
            return "You have no subscriptions."
        
        # Format results
        subscriptions = []
        for item in response['items']:
            snippet = item['snippet']
            subscriptions.append({
                'subscription_id': item['id'],
                'channel_id': snippet['resourceId']['channelId'],
                'channel_title': snippet['title'],
                'description': snippet['description'],
                'published_at': snippet['publishedAt'],
                'thumbnail': snippet['thumbnails']['default']['url']
            })
        
        output = f"Your Subscriptions ({len(subscriptions)} shown):\n\n"
        for i, sub in enumerate(subscriptions, 1):
            output += f"{i}. {sub['channel_title']}\n"
            output += f"   Channel ID: {sub['channel_id']}\n"
            output += f"   Subscribed Since: {sub['published_at']}\n"
            output += f"   Description: {sub['description'][:150]}...\n\n"
        
        total_results = response.get('pageInfo', {}).get('totalResults', len(subscriptions))
        if total_results > len(subscriptions):
            output += f"\nNote: You have {total_results} total subscriptions. "
            output += f"Use max_results parameter to see more.\n"
        
        return output
    
    except HttpError as e:
        return f"YouTube API Error: {e.resp.status} - {e.content.decode()}"
    except Exception as e:
        return f"Error: {str(e)}"


@mcp.tool()
def subscribe_to_channel(
    channel_id: str
) -> str:
    """
    Subscribe to a YouTube channel.
    
    Args:
        channel_id: YouTube channel ID to subscribe to
    
    Returns:
        Confirmation message
    """
    try:
        # Validate parameters
        if not channel_id or not channel_id.strip():
            return "Error: channel_id cannot be empty"
        
        # Get authenticated service
        youtube = get_authenticated_service()
        
        # Execute request
        request = youtube.subscriptions().insert(
            part='snippet',
            body={
                'snippet': {
                    'resourceId': {
                        'kind': 'youtube#channel',
                        'channelId': channel_id
                    }
                }
            }
        )
        response = request.execute()
        
        # Get channel title from response
        channel_title = response['snippet']['title']
        
        return f"✅ Successfully subscribed to channel: {channel_title} (ID: {channel_id})"
    
    except HttpError as e:
        error_content = e.content.decode()
        
        # Handle common errors
        if e.resp.status == 400:
            if 'subscriptionDuplicate' in error_content:
                return f"ℹ️ You are already subscribed to this channel (ID: {channel_id})"
            else:
                return f"Error: Invalid channel ID or request: {channel_id}"
        elif e.resp.status == 404:
            return f"Error: Channel not found: {channel_id}"
        else:
            return f"YouTube API Error: {e.resp.status} - {error_content}"
    
    except Exception as e:
        return f"Error: {str(e)}"


@mcp.tool()
def unsubscribe_from_channel(
    subscription_id: str
) -> str:
    """
    Unsubscribe from a YouTube channel.
    
    Args:
        subscription_id: Subscription ID (get this from list_subscriptions)
    
    Returns:
        Confirmation message
    
    Note:
        You need the subscription_id, not the channel_id. 
        Use list_subscriptions tool to get subscription IDs.
    """
    try:
        # Validate parameters
        if not subscription_id or not subscription_id.strip():
            return "Error: subscription_id cannot be empty"
        
        # Get authenticated service
        youtube = get_authenticated_service()
        
        # Execute request
        request = youtube.subscriptions().delete(
            id=subscription_id
        )
        request.execute()
        
        return f"✅ Successfully unsubscribed (Subscription ID: {subscription_id})"
    
    except HttpError as e:
        error_content = e.content.decode()
        
        # Handle common errors
        if e.resp.status == 404:
            return f"Error: Subscription not found: {subscription_id}"
        else:
            return f"YouTube API Error: {e.resp.status} - {error_content}"
    
    except Exception as e:
        return f"Error: {str(e)}"


# Run the server
if __name__ == "__main__":
    # Run with STDIO transport for local testing
    mcp.run()
```

### Step 2: Create Entry Point Script

Create `run_server.py` in the project root:

```python
#!/usr/bin/env python3
"""
Entry point for YouTube MCP Server
"""

from src.youtube_mcp.server import mcp

if __name__ == "__main__":
    mcp.run()
```

Make it executable:

```bash
chmod +x run_server.py
```

### Step 3: Create README

Create `README.md`:

```markdown
# YouTube MCP Server

A FastMCP server providing YouTube functionality through Model Context Protocol.

## Features

- 🔍 Search YouTube videos
- 📺 Get channel information
- 📋 List your subscriptions
- ➕ Subscribe to channels
- ➖ Unsubscribe from channels

## Setup

1. Install dependencies:
   ```bash
   uv sync
   ```

2. Place your `client_secret.json` in the project root

3. Run the server:
   ```bash
   python run_server.py
   ```

## Tools

### search_videos
Search for YouTube videos with customizable filters.

### get_channel_info
Retrieve detailed information about a YouTube channel.

### list_subscriptions
List your YouTube channel subscriptions.

### subscribe_to_channel
Subscribe to a YouTube channel.

### unsubscribe_from_channel
Unsubscribe from a YouTube channel.

## License

MIT
```

---

## Testing the Server

### Step 1: Test Authentication

```bash
# Activate virtual environment if not already active
source .venv/bin/activate  # macOS/Linux
# or
.venv\Scripts\activate  # Windows

# Run the server for first-time authentication
python run_server.py
```

**What happens:**
1. A browser window opens automatically
2. You'll see Google's OAuth consent screen
3. Sign in with your Google account
4. Grant permissions to the app
5. You'll see "The authentication flow has completed"
6. Close the browser
7. A `token.pickle` file is created (stores your credentials)

**Expected output:**
```
Server running with stdio transport
Press Ctrl+C to stop
```

### Step 2: Test with MCP Inspector

```bash
# Install MCP Inspector (in a separate terminal)
npm install -g @modelcontextprotocol/inspector

# Run inspector
npx @modelcontextprotocol/inspector python run_server.py
```

**What you'll see:**
1. Inspector opens in your browser at `http://localhost:5173`
2. Click **"Connect"**
3. You'll see 5 tools listed:
   - search_videos
   - get_channel_info
   - list_subscriptions
   - subscribe_to_channel
   - unsubscribe_from_channel

**Test each tool:**

**Example 1: Search Videos**
```json
{
  "query": "python tutorial",
  "max_results": 5
}
```

**Example 2: Get Channel Info**
```json
{
  "channel_username": "@GoogleDevelopers"
}
```

**Example 3: List Subscriptions**
```json
{
  "max_results": 10
}
```

### Step 3: Verify Tool Responses

Each tool should return formatted text with relevant information. If you see errors:

- **"No module named..."** → Run `uv sync` to install dependencies
- **"client_secret.json not found"** → Ensure file is in project root
- **"YouTube API Error: 403"** → Check that YouTube Data API v3 is enabled
- **"YouTube API Error: 401"** → Delete `token.pickle` and re-authenticate

---

## Integrating with Claude Desktop

### Step 1: Locate Claude Desktop Configuration

**macOS:**
```bash
# Configuration file location
~/Library/Application Support/Claude/claude_desktop_config.json
```

**Windows:**
```
%APPDATA%\Claude\claude_desktop_config.json
```

**Linux:**
```bash
~/.config/Claude/claude_desktop_config.json
```

### Step 2: Edit Configuration File

Open `claude_desktop_config.json` in a text editor:

```bash
# macOS
nano ~/Library/Application\ Support/Claude/claude_desktop_config.json

# Or use your preferred editor
code ~/Library/Application\ Support/Claude/claude_desktop_config.json
```

### Step 3: Add YouTube MCP Server Configuration

**Important:** Replace `/ABSOLUTE/PATH/TO/YOUR/PROJECT` with your actual project path.

```json
{
  "mcpServers": {
    "youtube": {
      "command": "python",
      "args": [
        "/ABSOLUTE/PATH/TO/YOUR/PROJECT/youtube-mcp-server/run_server.py"
      ],
      "env": {}
    }
  }
}
```

**Finding your absolute path:**

```bash
# In your project directory
pwd
# Copy the output and use it in the config above
```

**Example for macOS/Linux:**
```json
{
  "mcpServers": {
    "youtube": {
      "command": "python",
      "args": [
        "/Users/yourusername/projects/youtube-mcp-server/run_server.py"
      ],
      "env": {}
    }
  }
}
```

**Example for Windows:**
```json
{
  "mcpServers": {
    "youtube": {
      "command": "python",
      "args": [
        "C:\\Users\\YourUsername\\projects\\youtube-mcp-server\\run_server.py"
      ],
      "env": {}
    }
  }
}
```

**If you have multiple MCP servers:**
```json
{
  "mcpServers": {
    "youtube": {
      "command": "python",
      "args": [
        "/path/to/youtube-mcp-server/run_server.py"
      ],
      "env": {}
    },
    "other-server": {
      "command": "node",
      "args": [
        "/path/to/other-server/index.js"
      ]
    }
  }
}
```

### Step 4: Restart Claude Desktop

1. **Completely quit** Claude Desktop (not just close the window)
   - macOS: `Cmd + Q`
   - Windows: Right-click taskbar icon → Quit
   - Linux: Close application completely

2. **Relaunch** Claude Desktop

3. Wait ~5 seconds for servers to initialize

### Step 5: Verify Integration

1. Open a new conversation in Claude Desktop

2. Look for the **tools icon** (hammer/wrench) in the message input area

3. Click the tools icon to see available tools

4. You should see 5 YouTube tools:
   - 🔍 search_videos
   - 📺 get_channel_info
   - 📋 list_subscriptions
   - ➕ subscribe_to_channel
   - ➖ unsubscribe_from_channel

**If tools don't appear:**
- Check Claude Desktop logs:
  - macOS: `~/Library/Logs/Claude/`
  - Windows: `%APPDATA%\Claude\logs\`
- Verify the path in config is absolute and correct
- Ensure `run_server.py` has execute permissions
- Check that virtual environment has all dependencies

---

## Usage Examples

### Example 1: Search for Python Tutorials

**You say to Claude:**
```
Search YouTube for the top 5 Python tutorial videos for beginners
```

**Claude's response:**
```
I'll search for Python tutorial videos for beginners.

[Claude calls search_videos tool with query="python tutorial for beginners", max_results=5]

Found 5 videos for 'python tutorial for beginners':

1. Python Tutorial for Beginners - Learn Python in 5 Hours
   Channel: Programming with Mosh
   URL: https://www.youtube.com/watch?v=...
   Published: 2023-05-15T10:30:00Z
   Description: Complete Python tutorial for absolute beginners...

[... 4 more results ...]
```

### Example 2: Get Information About a Channel

**You say to Claude:**
```
Tell me about the @GoogleDevelopers YouTube channel
```

**Claude's response:**
```
Let me get information about the Google Developers channel.

[Claude calls get_channel_info tool with channel_username="@GoogleDevelopers"]

Channel Information:

Title: Google Developers
Channel ID: UC_x5XG1OV2P6uZZ5FSM9Ttw
Description: The Google Developers channel features talks from events, educational series...
Custom URL: @GoogleDevelopers
Published At: 2007-08-23T00:34:43Z
Country: United States

Statistics:
  Subscribers: 2,350,000
  Total Views: 305,234,567
  Total Videos: 6,892

Channel URL: https://www.youtube.com/channel/UC_x5XG1OV2P6uZZ5FSM9Ttw
```

### Example 3: Manage Subscriptions

**You say to Claude:**
```
Show me my YouTube subscriptions
```

**Claude's response:**
```
I'll list your YouTube subscriptions.

[Claude calls list_subscriptions tool with max_results=25]

Your Subscriptions (15 shown):

1. Google Developers
   Channel ID: UC_x5XG1OV2P6uZZ5FSM9Ttw
   Subscribed Since: 2020-03-15T14:20:00Z
   Description: The Google Developers channel features...

2. Python
   Channel ID: UCXv6JNs8hMOYW6t8P1tQr4w
   Subscribed Since: 2019-11-20T09:15:00Z
   Description: Official Python Software Foundation channel...

[... 13 more subscriptions ...]
```

**You say to Claude:**
```
Subscribe me to the Python channel with ID UCXv6JNs8hMOYW6t8P1tQr4w
```

**Claude's response:**
```
[Claude calls subscribe_to_channel tool with channel_id="UCXv6JNs8hMOYW6t8P1tQr4w"]

✅ Successfully subscribed to channel: Python (ID: UCXv6JNs8hMOYW6t8P1tQr4w)
```

### Example 4: Complex Multi-Tool Workflow

**You say to Claude:**
```
Search for the best FastAPI tutorials, find the channel with the most subscribers, 
and subscribe me to that channel
```

**Claude's response:**
```
I'll help you find FastAPI tutorials and subscribe to the most popular channel.

Step 1: Searching for FastAPI tutorials...
[Claude calls search_videos with query="FastAPI tutorial"]

Found 10 videos. The top channels are:
1. ArjanCodes - Multiple tutorials
2. Pretty Printed - FastAPI series
3. Bitfumes - FastAPI course

Step 2: Checking subscriber counts...
[Claude calls get_channel_info for each channel]

ArjanCodes: 450,000 subscribers
Pretty Printed: 180,000 subscribers
Bitfumes: 120,000 subscribers

The most subscribed channel is ArjanCodes with 450,000 subscribers.

Step 3: Subscribing to ArjanCodes...
[Claude calls subscribe_to_channel with ArjanCodes channel ID]

✅ Successfully subscribed to channel: ArjanCodes

You're now subscribed to the most popular FastAPI tutorial channel!
```

### Example 5: Research and Discovery

**You say to Claude:**
```
I want to learn web development. Search for web development tutorials, 
check the top 3 channels, and tell me which one is best for beginners 
based on their descriptions and video count
```

**Claude uses multiple tools to:**
1. Search for web development tutorials
2. Extract channel IDs from top results
3. Get detailed info for top 3 channels
4. Analyze and recommend based on content

---

## Troubleshooting

### Issue 1: Server Won't Start

**Error:** `ModuleNotFoundError: No module named 'fastmcp'`

**Solution:**
```bash
# Activate virtual environment
source .venv/bin/activate  # macOS/Linux
.venv\Scripts\activate     # Windows

# Install dependencies
uv sync
```

---

### Issue 2: Authentication Fails

**Error:** `client_secret.json not found`

**Solution:**
```bash
# Verify file exists
ls -la client_secret.json

# If not, download from Google Cloud Console:
# 1. Go to APIs & Services → Credentials
# 2. Click OAuth 2.0 Client ID
# 3. Download JSON
# 4. Save as client_secret.json in project root
```

**Error:** `redirect_uri_mismatch`

**Solution:**
OAuth client type must be "Desktop app", not "Web application". Create a new OAuth client with the correct type.

---

### Issue 3: API Quota Exceeded

**Error:** `YouTube API Error: 403 - quotaExceeded`

**Solution:**
- You've exceeded the daily 10,000 unit quota
- Wait until quota resets (midnight Pacific Time)
- Or request quota increase: https://developers.google.com/youtube/v3/determine_quota_cost

**Quota costs:**
- search_videos: 100 units per call
- get_channel_info: 1 unit per call
- list_subscriptions: 1 unit per call
- subscribe_to_channel: 50 units per call
- unsubscribe_from_channel: 50 units per call

---

### Issue 4: Claude Desktop Doesn't See Tools

**Problem:** Tools don't appear in Claude Desktop

**Solutions:**

1. **Check config path is absolute:**
   ```bash
   # Get absolute path
   cd /path/to/youtube-mcp-server
   pwd
   # Use this FULL path in claude_desktop_config.json
   ```

2. **Verify config syntax:**
   - Use a JSON validator: https://jsonlint.com/
   - Ensure commas are correct
   - Ensure paths use forward slashes (/) or escaped backslashes (\\)

3. **Check Claude Desktop logs:**
   ```bash
   # macOS
   tail -f ~/Library/Logs/Claude/mcp*.log
   
   # Windows
   type %APPDATA%\Claude\logs\mcp*.log
   ```

4. **Restart Claude Desktop completely:**
   - Don't just close the window
   - Quit the application fully
   - Relaunch

---

### Issue 5: Permission Denied Errors

**Error:** `[Errno 13] Permission denied: '/path/to/token.pickle'`

**Solution:**
```bash
# Make sure you own the project directory
sudo chown -R $USER:$USER /path/to/youtube-mcp-server

# Ensure run_server.py is executable
chmod +x run_server.py
```

---

## Conclusion

You now have a fully functional YouTube MCP server that works with Claude Desktop! You can:

✅ Search YouTube videos  
✅ Get channel information  
✅ Manage subscriptions  
✅ Interact naturally with YouTube through Claude  
✅ Build more complex workflows combining multiple tools  

### Next Steps

1. **Experiment with different queries** in Claude Desktop
2. **Add more YouTube API endpoints** following the pattern
3. **Share your server** with the MCP community
4. **Build automation workflows** using Claude's agentic capabilities

Happy building! 🚀
