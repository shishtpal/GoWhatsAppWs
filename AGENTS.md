# whatsmeow - Go WhatsApp Web API Library

## Overview

**whatsmeow** is a Go library implementing the WhatsApp Web multidevice API. It enables building WhatsApp bots and integrations by providing a complete client for the WhatsApp protocol, including end-to-end encryption via the Signal protocol.

- **Module**: `go.mau.fi/whatsmeow`
- **Author**: Tulir Asokan
- **License**: Mozilla Public License 2.0
- **Go Version**: 1.24+

## Architecture

```
whatsmeow/
├── client.go           # Main Client struct and connection logic
├── send.go             # Message sending functionality
├── download.go         # Media download and decryption
├── upload.go           # Media upload and encryption
├── group.go            # Group management operations
├── user.go             # User profile operations
├── presence.go         # Presence/typing indicators
├── receipt.go          # Read receipts handling
├── appstate/           # App state synchronization (contacts, settings)
├── binary/             # WhatsApp binary protocol encoding/decoding
├── proto/              # Protocol buffer definitions
├── socket/             # WebSocket and Noise protocol handling
├── store/              # Session and data persistence
│   └── sqlstore/       # SQL-based storage implementation
├── types/              # Common types (JID, MessageInfo, etc.)
│   └── events/         # Event types emitted by the client
└── util/               # Utility packages (crypto, logging)
```

## Core Components

### 1. Client (`client.go`)

The `Client` struct is the main entry point for interacting with WhatsApp.

```go
type Client struct {
    Store   *store.Device    // Device credentials and session data
    Log     waLog.Logger     // Logger instance
    
    // Configuration options
    EnableAutoReconnect   bool
    AutoTrustIdentity     bool
    SynchronousAck        bool
    
    // Callbacks
    GetMessageForRetry    func(requester, to types.JID, id types.MessageID) *waE2E.Message
    PrePairCallback       func(jid types.JID, platform, businessName string) bool
    AutoReconnectHook     func(error) bool
}
```

### 2. Device Store (`store/`)

Handles persistence of session data, encryption keys, and contacts.

```go
type Device struct {
    ID              *types.JID       // Device JID
    RegistrationID  uint32           // Signal registration ID
    NoiseKey        *keys.KeyPair    // Noise protocol keys
    IdentityKey     *keys.KeyPair    // Signal identity keys
    SignedPreKey    *keys.PreKey     // Signal signed pre-key
    
    // Storage interfaces
    Sessions        SessionStore
    PreKeys         PreKeyStore
    SenderKeys      SenderKeyStore
    Contacts        ContactStore
    // ... more stores
}
```

### 3. Types (`types/`)

#### JID (Jabber ID)
Represents WhatsApp user/group identifiers.

```go
type JID struct {
    User       string   // Phone number or group ID
    RawAgent   uint8    // Device agent
    Device     uint16   // Device number
    Server     string   // Server domain
}

// Server constants
const (
    DefaultUserServer = "s.whatsapp.net"  // Regular users
    GroupServer       = "g.us"             // Groups
    BroadcastServer   = "broadcast"        // Broadcast lists
    NewsletterServer  = "newsletter"       // Channels
)
```

#### MessageInfo
Contains metadata about a message.

```go
type MessageInfo struct {
    MessageSource
    ID        MessageID      // Unique message identifier
    ServerID  MessageServerID
    Type      string         // Message type
    PushName  string         // Sender's push name
    Timestamp time.Time      // Message timestamp
    Category  string         // Message category
}

type MessageSource struct {
    Chat     JID   // Chat where message was sent
    Sender   JID   // Message sender
    IsFromMe bool  // Whether message is from current user
    IsGroup  bool  // Whether message is in a group
}
```

### 4. Events (`types/events/`)

Events are emitted via registered event handlers.

```go
// Core events
type QR struct { Codes []string }           // QR code for pairing
type PairSuccess struct { ID, LID types.JID } // Successful pairing
type Connected struct{}                      // Connection established
type Disconnected struct{}                   // Connection lost
type LoggedOut struct { Reason ConnectFailureReason }

// Message events
type Message struct {
    Info       types.MessageInfo
    Message    *waE2E.Message
    RawMessage *waE2E.Message
    IsEphemeral, IsViewOnce, IsEdit bool
}

type Receipt struct {
    types.MessageSource
    MessageIDs []types.MessageID
    Timestamp  time.Time
    Type       types.ReceiptType  // delivered, read, played
}

// Group events
type JoinedGroup struct { types.GroupInfo }
type GroupInfo struct {
    JID       types.JID
    Name      *types.GroupName
    Topic     *types.GroupTopic
    Join      []types.JID
    Leave     []types.JID
    Promote   []types.JID
    Demote    []types.JID
}

// Presence events
type Presence struct {
    From        types.JID
    Unavailable bool
    LastSeen    time.Time
}

type ChatPresence struct {
    types.MessageSource
    State types.ChatPresence  // composing, paused
    Media types.ChatPresenceMedia
}
```

---

## Usage Examples

### Basic Setup and Connection

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"

    _ "github.com/mattn/go-sqlite3"
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/store/sqlstore"
    "go.mau.fi/whatsmeow/types/events"
    waLog "go.mau.fi/whatsmeow/util/log"
)

func main() {
    // Initialize logger
    logger := waLog.Stdout("Main", "DEBUG", true)

    // Create database container
    container, err := sqlstore.New(context.Background(), "sqlite3", 
        "file:whatsapp.db?_foreign_keys=on", nil)
    if err != nil {
        panic(err)
    }

    // Get or create device
    deviceStore, err := container.GetFirstDevice(context.Background())
    if err != nil {
        panic(err)
    }

    // Create client
    client := whatsmeow.NewClient(deviceStore, logger)
    
    // Register event handler
    client.AddEventHandler(eventHandler)

    // Connect
    if client.Store.ID == nil {
        // Need to pair - QR code will be emitted via event
        qrChan, _ := client.GetQRChannel(context.Background())
        err = client.Connect()
        if err != nil {
            panic(err)
        }
        for evt := range qrChan {
            if evt.Event == "code" {
                fmt.Println("QR code:", evt.Code)
                // Display this QR code to user
            }
        }
    } else {
        // Already paired
        err = client.Connect()
        if err != nil {
            panic(err)
        }
    }

    // Wait for interrupt
    c := make(chan os.Signal, 1)
    signal.Notify(c, os.Interrupt, syscall.SIGTERM)
    <-c

    client.Disconnect()
}

func eventHandler(evt interface{}) {
    switch v := evt.(type) {
    case *events.Connected:
        fmt.Println("Connected!")
    case *events.Message:
        fmt.Printf("Message from %s: %s\n", v.Info.Sender, v.Message.GetConversation())
    case *events.Receipt:
        fmt.Printf("Receipt: %v - %s\n", v.MessageIDs, v.Type)
    }
}
```

### Sending Messages

#### Text Messages

```go
import (
    "context"
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/proto/waE2E"
    "go.mau.fi/whatsmeow/types"
    "google.golang.org/protobuf/proto"
)

// Simple text message
func sendTextMessage(client *whatsmeow.Client, to string, text string) error {
    jid, err := types.ParseJID(to)
    if err != nil {
        return err
    }
    
    _, err = client.SendMessage(context.Background(), jid, &waE2E.Message{
        Conversation: proto.String(text),
    })
    return err
}

// Extended text with formatting and mentions
func sendExtendedText(client *whatsmeow.Client, to types.JID, text string, mentions []string) error {
    mentionedJIDs := make([]string, len(mentions))
    for i, m := range mentions {
        jid, _ := types.ParseJID(m)
        mentionedJIDs[i] = jid.String()
    }
    
    _, err := client.SendMessage(context.Background(), to, &waE2E.Message{
        ExtendedTextMessage: &waE2E.ExtendedTextMessage{
            Text: proto.String(text),
            ContextInfo: &waE2E.ContextInfo{
                MentionedJID: mentionedJIDs,
            },
        },
    })
    return err
}

// Reply to a message
func sendReply(client *whatsmeow.Client, to types.JID, replyToID string, replyToSender types.JID, text string) error {
    _, err := client.SendMessage(context.Background(), to, &waE2E.Message{
        ExtendedTextMessage: &waE2E.ExtendedTextMessage{
            Text: proto.String(text),
            ContextInfo: &waE2E.ContextInfo{
                StanzaID:      proto.String(replyToID),
                Participant:   proto.String(replyToSender.String()),
                QuotedMessage: &waE2E.Message{Conversation: proto.String("original message")},
            },
        },
    })
    return err
}
```

#### Media Messages

```go
import (
    "context"
    "os"
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/proto/waE2E"
    "go.mau.fi/whatsmeow/types"
    "google.golang.org/protobuf/proto"
)

// Send image
func sendImage(client *whatsmeow.Client, to types.JID, imagePath, caption string) error {
    data, err := os.ReadFile(imagePath)
    if err != nil {
        return err
    }
    
    // Upload to WhatsApp servers
    resp, err := client.Upload(context.Background(), data, whatsmeow.MediaImage)
    if err != nil {
        return err
    }
    
    // Send message with uploaded media
    _, err = client.SendMessage(context.Background(), to, &waE2E.Message{
        ImageMessage: &waE2E.ImageMessage{
            Caption:       proto.String(caption),
            Mimetype:      proto.String("image/jpeg"),
            URL:           proto.String(resp.URL),
            DirectPath:    proto.String(resp.DirectPath),
            MediaKey:      resp.MediaKey,
            FileEncSHA256: resp.FileEncSHA256,
            FileSHA256:    resp.FileSHA256,
            FileLength:    proto.Uint64(resp.FileLength),
        },
    })
    return err
}

// Send document
func sendDocument(client *whatsmeow.Client, to types.JID, filePath, filename, mimetype string) error {
    data, err := os.ReadFile(filePath)
    if err != nil {
        return err
    }
    
    resp, err := client.Upload(context.Background(), data, whatsmeow.MediaDocument)
    if err != nil {
        return err
    }
    
    _, err = client.SendMessage(context.Background(), to, &waE2E.Message{
        DocumentMessage: &waE2E.DocumentMessage{
            Title:         proto.String(filename),
            FileName:      proto.String(filename),
            Mimetype:      proto.String(mimetype),
            URL:           proto.String(resp.URL),
            DirectPath:    proto.String(resp.DirectPath),
            MediaKey:      resp.MediaKey,
            FileEncSHA256: resp.FileEncSHA256,
            FileSHA256:    resp.FileSHA256,
            FileLength:    proto.Uint64(resp.FileLength),
        },
    })
    return err
}

// Send audio (voice note)
func sendVoiceNote(client *whatsmeow.Client, to types.JID, audioPath string, durationSeconds uint32) error {
    data, err := os.ReadFile(audioPath)
    if err != nil {
        return err
    }
    
    resp, err := client.Upload(context.Background(), data, whatsmeow.MediaAudio)
    if err != nil {
        return err
    }
    
    _, err = client.SendMessage(context.Background(), to, &waE2E.Message{
        AudioMessage: &waE2E.AudioMessage{
            Mimetype:      proto.String("audio/ogg; codecs=opus"),
            URL:           proto.String(resp.URL),
            DirectPath:    proto.String(resp.DirectPath),
            MediaKey:      resp.MediaKey,
            FileEncSHA256: resp.FileEncSHA256,
            FileSHA256:    resp.FileSHA256,
            FileLength:    proto.Uint64(resp.FileLength),
            Seconds:       proto.Uint32(durationSeconds),
            PTT:           proto.Bool(true), // Push-to-talk (voice note)
        },
    })
    return err
}

// Send video
func sendVideo(client *whatsmeow.Client, to types.JID, videoPath, caption string) error {
    data, err := os.ReadFile(videoPath)
    if err != nil {
        return err
    }
    
    resp, err := client.Upload(context.Background(), data, whatsmeow.MediaVideo)
    if err != nil {
        return err
    }
    
    _, err = client.SendMessage(context.Background(), to, &waE2E.Message{
        VideoMessage: &waE2E.VideoMessage{
            Caption:       proto.String(caption),
            Mimetype:      proto.String("video/mp4"),
            URL:           proto.String(resp.URL),
            DirectPath:    proto.String(resp.DirectPath),
            MediaKey:      resp.MediaKey,
            FileEncSHA256: resp.FileEncSHA256,
            FileSHA256:    resp.FileSHA256,
            FileLength:    proto.Uint64(resp.FileLength),
        },
    })
    return err
}
```

#### Download Media

```go
import (
    "context"
    "os"
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/types/events"
)

func handleMediaMessage(client *whatsmeow.Client, msg *events.Message) error {
    // Download image
    if img := msg.Message.GetImageMessage(); img != nil {
        data, err := client.Download(context.Background(), img)
        if err != nil {
            return err
        }
        return os.WriteFile("downloaded_image.jpg", data, 0644)
    }
    
    // Download document
    if doc := msg.Message.GetDocumentMessage(); doc != nil {
        data, err := client.Download(context.Background(), doc)
        if err != nil {
            return err
        }
        return os.WriteFile(doc.GetFileName(), data, 0644)
    }
    
    // Download video
    if vid := msg.Message.GetVideoMessage(); vid != nil {
        data, err := client.Download(context.Background(), vid)
        if err != nil {
            return err
        }
        return os.WriteFile("downloaded_video.mp4", data, 0644)
    }
    
    // Download audio
    if audio := msg.Message.GetAudioMessage(); audio != nil {
        data, err := client.Download(context.Background(), audio)
        if err != nil {
            return err
        }
        ext := ".ogg"
        if !audio.GetPTT() {
            ext = ".mp3"
        }
        return os.WriteFile("downloaded_audio"+ext, data, 0644)
    }
    
    return nil
}
```

### Reactions and Message Actions

```go
// Send reaction
func sendReaction(client *whatsmeow.Client, chat, sender types.JID, messageID, emoji string) error {
    _, err := client.SendMessage(context.Background(), chat, 
        client.BuildReaction(chat, sender, messageID, emoji))
    return err
}

// Remove reaction
func removeReaction(client *whatsmeow.Client, chat, sender types.JID, messageID string) error {
    _, err := client.SendMessage(context.Background(), chat, 
        client.BuildReaction(chat, sender, messageID, "")) // empty string removes reaction
    return err
}

// Delete/revoke message (for everyone)
func revokeMessage(client *whatsmeow.Client, chat types.JID, messageID string) error {
    _, err := client.SendMessage(context.Background(), chat, 
        client.BuildRevoke(chat, types.EmptyJID, messageID))
    return err
}

// Edit message (within 20 minutes)
func editMessage(client *whatsmeow.Client, chat types.JID, messageID, newText string) error {
    _, err := client.SendMessage(context.Background(), chat, 
        client.BuildEdit(chat, messageID, &waE2E.Message{
            Conversation: proto.String(newText),
        }))
    return err
}
```

### Group Management

```go
import (
    "context"
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/types"
)

// Create a new group
func createGroup(client *whatsmeow.Client, name string, participants []types.JID) (*types.GroupInfo, error) {
    return client.CreateGroup(context.Background(), whatsmeow.ReqCreateGroup{
        Name:         name,
        Participants: participants,
    })
}

// Get group info
func getGroupInfo(client *whatsmeow.Client, groupJID types.JID) (*types.GroupInfo, error) {
    return client.GetGroupInfo(context.Background(), groupJID)
}

// Get all joined groups
func getJoinedGroups(client *whatsmeow.Client) ([]*types.GroupInfo, error) {
    return client.GetJoinedGroups(context.Background())
}

// Add participants to group
func addParticipants(client *whatsmeow.Client, groupJID types.JID, participants []types.JID) error {
    _, err := client.UpdateGroupParticipants(context.Background(), groupJID, 
        participants, whatsmeow.ParticipantChangeAdd)
    return err
}

// Remove participants from group
func removeParticipants(client *whatsmeow.Client, groupJID types.JID, participants []types.JID) error {
    _, err := client.UpdateGroupParticipants(context.Background(), groupJID, 
        participants, whatsmeow.ParticipantChangeRemove)
    return err
}

// Promote to admin
func promoteToAdmin(client *whatsmeow.Client, groupJID types.JID, participants []types.JID) error {
    _, err := client.UpdateGroupParticipants(context.Background(), groupJID, 
        participants, whatsmeow.ParticipantChangePromote)
    return err
}

// Demote from admin
func demoteFromAdmin(client *whatsmeow.Client, groupJID types.JID, participants []types.JID) error {
    _, err := client.UpdateGroupParticipants(context.Background(), groupJID, 
        participants, whatsmeow.ParticipantChangeDemote)
    return err
}

// Leave group
func leaveGroup(client *whatsmeow.Client, groupJID types.JID) error {
    return client.LeaveGroup(context.Background(), groupJID)
}

// Update group name
func setGroupName(client *whatsmeow.Client, groupJID types.JID, name string) error {
    return client.SetGroupName(context.Background(), groupJID, name)
}

// Update group description
func setGroupDescription(client *whatsmeow.Client, groupJID types.JID, description string) error {
    return client.SetGroupTopic(context.Background(), groupJID, "", "", description)
}

// Get invite link
func getGroupInviteLink(client *whatsmeow.Client, groupJID types.JID) (string, error) {
    return client.GetGroupInviteLink(context.Background(), groupJID, false)
}

// Reset/revoke invite link
func resetGroupInviteLink(client *whatsmeow.Client, groupJID types.JID) (string, error) {
    return client.GetGroupInviteLink(context.Background(), groupJID, true)
}

// Join group via invite link
func joinGroupViaLink(client *whatsmeow.Client, inviteLink string) (types.JID, error) {
    return client.JoinGroupWithLink(context.Background(), inviteLink)
}

// Get group info from invite link (without joining)
func getGroupInfoFromLink(client *whatsmeow.Client, inviteLink string) (*types.GroupInfo, error) {
    return client.GetGroupInfoFromLink(context.Background(), inviteLink)
}

// Set group to admin-only messages
func setGroupAnnounce(client *whatsmeow.Client, groupJID types.JID, announce bool) error {
    return client.SetGroupAnnounce(context.Background(), groupJID, announce)
}

// Set group to admin-only edit info
func setGroupLocked(client *whatsmeow.Client, groupJID types.JID, locked bool) error {
    return client.SetGroupLocked(context.Background(), groupJID, locked)
}

// Set disappearing messages timer
func setDisappearingMessages(client *whatsmeow.Client, jid types.JID, timer time.Duration) error {
    // Valid timers: 0 (off), 24h, 7d, 90d
    return client.SetDisappearingTimer(context.Background(), jid, timer, time.Time{})
}
```

### Presence and Typing

```go
import (
    "context"
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/types"
)

// Set online/offline status
func setPresence(client *whatsmeow.Client, available bool) error {
    if available {
        return client.SendPresence(types.PresenceAvailable)
    }
    return client.SendPresence(types.PresenceUnavailable)
}

// Send typing indicator
func sendTyping(client *whatsmeow.Client, to types.JID, composing bool) error {
    if composing {
        return client.SendChatPresence(to, types.ChatPresenceComposing, types.ChatPresenceMediaText)
    }
    return client.SendChatPresence(to, types.ChatPresencePaused, types.ChatPresenceMediaText)
}

// Send recording audio indicator
func sendRecording(client *whatsmeow.Client, to types.JID) error {
    return client.SendChatPresence(to, types.ChatPresenceComposing, types.ChatPresenceMediaAudio)
}

// Subscribe to user's presence (online/offline status)
func subscribePresence(client *whatsmeow.Client, userJID types.JID) error {
    return client.SubscribePresence(userJID)
}
```

### User Information

```go
import (
    "context"
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/types"
)

// Check if numbers are on WhatsApp
func checkOnWhatsApp(client *whatsmeow.Client, phones []string) ([]types.IsOnWhatsAppResponse, error) {
    return client.IsOnWhatsApp(phones)
}

// Get user info (including profile picture info)
func getUserInfo(client *whatsmeow.Client, jids []types.JID) (map[types.JID]types.UserInfo, error) {
    return client.GetUserInfo(context.Background(), jids)
}

// Get profile picture
func getProfilePicture(client *whatsmeow.Client, jid types.JID) (*types.ProfilePictureInfo, error) {
    return client.GetProfilePictureInfo(context.Background(), jid, nil)
}

// Set own profile picture
func setProfilePicture(client *whatsmeow.Client, imageData []byte) (string, error) {
    return client.SetGroupPhoto(context.Background(), types.EmptyJID, imageData)
}

// Get user devices (for multi-device)
func getUserDevices(client *whatsmeow.Client, jids []types.JID) ([]types.JID, error) {
    return client.GetUserDevices(context.Background(), jids)
}
```

### Receipts

```go
import (
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/types"
    "go.mau.fi/whatsmeow/types/events"
)

// Mark message as read
func markRead(client *whatsmeow.Client, msg *events.Message) error {
    return client.MarkRead([]types.MessageID{msg.Info.ID}, msg.Info.Timestamp, 
        msg.Info.Chat, msg.Info.Sender)
}

// Control read receipt sending behavior
func configureReadReceipts(client *whatsmeow.Client) {
    // Don't send read receipts automatically
    client.SetSendActiveReceipts(types.SendReceiptNone)
    
    // Send read receipts only in private chats
    // client.SetSendActiveReceipts(types.SendReceiptOnlyPrivate)
    
    // Send read receipts everywhere (default)
    // client.SetSendActiveReceipts(types.SendReceiptAll)
}
```

### Complete Bot Example

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "strings"
    "syscall"

    _ "github.com/mattn/go-sqlite3"
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/proto/waE2E"
    "go.mau.fi/whatsmeow/store/sqlstore"
    "go.mau.fi/whatsmeow/types"
    "go.mau.fi/whatsmeow/types/events"
    waLog "go.mau.fi/whatsmeow/util/log"
    "google.golang.org/protobuf/proto"
)

var client *whatsmeow.Client

func main() {
    logger := waLog.Stdout("Bot", "INFO", true)
    
    container, err := sqlstore.New(context.Background(), "sqlite3", 
        "file:bot.db?_foreign_keys=on", nil)
    if err != nil {
        panic(err)
    }
    
    device, err := container.GetFirstDevice(context.Background())
    if err != nil {
        panic(err)
    }
    
    client = whatsmeow.NewClient(device, logger)
    client.AddEventHandler(handleEvent)
    
    if client.Store.ID == nil {
        qrChan, _ := client.GetQRChannel(context.Background())
        if err := client.Connect(); err != nil {
            panic(err)
        }
        for evt := range qrChan {
            if evt.Event == "code" {
                fmt.Println("Scan QR:", evt.Code)
            }
        }
    } else {
        if err := client.Connect(); err != nil {
            panic(err)
        }
    }
    
    c := make(chan os.Signal, 1)
    signal.Notify(c, os.Interrupt, syscall.SIGTERM)
    <-c
    client.Disconnect()
}

func handleEvent(evt interface{}) {
    switch v := evt.(type) {
    case *events.Connected:
        fmt.Println("Bot connected!")
        
    case *events.Message:
        handleMessage(v)
        
    case *events.Receipt:
        if v.Type == types.ReceiptTypeRead {
            fmt.Printf("Message %v read by %s\n", v.MessageIDs, v.Sender)
        }
        
    case *events.Presence:
        status := "online"
        if v.Unavailable {
            status = "offline"
        }
        fmt.Printf("%s is %s\n", v.From, status)
        
    case *events.GroupInfo:
        fmt.Printf("Group %s updated\n", v.JID)
    }
}

func handleMessage(msg *events.Message) {
    // Ignore own messages
    if msg.Info.IsFromMe {
        return
    }
    
    text := msg.Message.GetConversation()
    if text == "" {
        text = msg.Message.GetExtendedTextMessage().GetText()
    }
    
    if text == "" {
        return
    }
    
    fmt.Printf("[%s] %s: %s\n", msg.Info.Chat, msg.Info.Sender, text)
    
    // Command handling
    switch {
    case strings.HasPrefix(text, "!ping"):
        reply(msg.Info.Chat, "Pong!")
        
    case strings.HasPrefix(text, "!help"):
        reply(msg.Info.Chat, "Available commands:\n!ping - Check bot\n!help - Show help")
        
    case strings.HasPrefix(text, "!info"):
        if msg.Info.IsGroup {
            info, err := client.GetGroupInfo(context.Background(), msg.Info.Chat)
            if err != nil {
                reply(msg.Info.Chat, "Error: "+err.Error())
                return
            }
            reply(msg.Info.Chat, fmt.Sprintf("Group: %s\nParticipants: %d\nCreated: %s",
                info.Name, len(info.Participants), info.GroupCreated.String()))
        }
    }
}

func reply(to types.JID, text string) {
    _, err := client.SendMessage(context.Background(), to, &waE2E.Message{
        Conversation: proto.String(text),
    })
    if err != nil {
        fmt.Println("Error sending:", err)
    }
}
```

---

## Important Notes

### Message ID Generation
```go
// Generate proper message ID (recommended)
msgID := client.GenerateMessageID()

// Use custom ID in send
client.SendMessage(ctx, to, message, whatsmeow.SendRequestExtra{
    ID: msgID,
})
```

### Error Handling
```go
import "go.mau.fi/whatsmeow"

// Common errors to handle
var (
    ErrNotLoggedIn          = whatsmeow.ErrNotLoggedIn
    ErrNotConnected         = whatsmeow.ErrNotConnected
    ErrMessageTimedOut      = whatsmeow.ErrMessageTimedOut
    ErrMediaDownloadFailed  = whatsmeow.ErrMediaDownloadFailedWith404
    ErrGroupNotFound        = whatsmeow.ErrGroupNotFound
    ErrNotInGroup           = whatsmeow.ErrNotInGroup
)
```

### Proxy Configuration
```go
// HTTP proxy
client.SetProxyAddress("http://proxy.example.com:8080")

// SOCKS5 proxy
client.SetProxyAddress("socks5://proxy.example.com:1080")

// Different proxy for media vs websocket
client.SetProxy(func(r *http.Request) (*url.URL, error) {
    if r.URL.Host == "web.whatsapp.com" {
        return websocketProxy, nil
    }
    return mediaProxy, nil
})
```

### Session Management
```go
// Logout and clear session
func logout(client *whatsmeow.Client) error {
    err := client.Logout(context.Background())
    if err != nil {
        return err
    }
    return client.Store.Delete(context.Background())
}

// Get all sessions
func getAllSessions(container *sqlstore.Container) ([]*store.Device, error) {
    return container.GetAllDevices(context.Background())
}
```

---

## Protocol Details

### JID Formats
- User: `1234567890@s.whatsapp.net`
- Group: `123456789012345678@g.us`
- Device: `1234567890.0:1@s.whatsapp.net`
- Newsletter: `123456789012345678@newsletter`
- Broadcast: `status@broadcast`

### Message Types
- `text` - Text messages
- `media` - Images, videos, documents, audio
- `poll` - Poll messages
- `reaction` - Message reactions

### Receipt Types
- `delivered` - Message delivered
- `read` - Message read
- `played` - Audio/video played
- `sender` - Receipt from own devices

---

## Features Status

**Implemented:**
- Sending/receiving all message types
- Group management
- Media upload/download
- Presence and typing indicators
- Read receipts
- Message reactions
- Message editing and deletion
- App state sync
- History sync
- Newsletters/Channels

**Not Implemented:**
- Voice/video calls
- Broadcast list messages (not supported by WhatsApp Web)

---

## Resources

- **Documentation**: https://pkg.go.dev/go.mau.fi/whatsmeow
- **Matrix Room**: #whatsmeow:maunium.net
- **GitHub**: https://github.com/tulir/whatsmeow
- **WhatsApp Protocol Q&A**: GitHub Discussions
