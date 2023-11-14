+++
title = 'Mirror-Networking Character Creator'
date = 2023-11-13T21:56:14-04:00
+++

I am working on a little multiplayer Unity game and decided to use [Mirror Networking](https://mirror-networking.com/) for handling the server and communication. In this post I'm going to outline how I use Mirror to build a simple Character Builder and syncing that between clients.

## Mirror Concept Breakdown

We're going to leverage the [NetworkRoomManger](https://mirror-networking.gitbook.io/docs/manual/components/network-room-manager) for our game. The NetworkRoomManager handles the flow of a player connecting or becoming the host, create a room, ensure players are ready, transitioning to the game scene, creating your character prefab and along the way offering hooks for customing the different steps.

As you can see, NetworkRoomManager does a lot for us and only asks for a few objects to be configured in return; An offline scene, an online, your transport, a player prefab, a room player prefab, a room scene and a game scene.

Example:

![NetworkRoomManagerExt Config](/mirror-character-creator/NetworkRoomManagerExt_Example.png)

Let's break these requirements down

- Offline scene
  - This is the scene that will contain your Monobehaviour that has your NetworkRoomManagerExt.cs attached. This is the the scene to boot your server. Likely your menu.
- Online Scene
  - This is the scene that the player transitions to once they join/host a game.
- Transport
  - This is the most straightforward. Make a choice on your protocal and set it.
- Player Prefab
  - ![Whacker_Controller_VO](/mirror-character-creator/Whacker_Controller_VO.png)
  - Note WhackerController and PlayerSyncVO - These are NetworkBehaviour and are capiable of syncing across the server
- Room Player Prefab
  - ![Whacker_Controller_VO](/mirror-character-creator/RoomPlayerPrefab.png)
  - This can be an empty prefab as long as it has attached your classes extending NetworkRoomPlayer.
  - Is used to set/sync data before all players are ready. We will use this to set the selected character details.
- Room Scene
  - Can be it's own thing, but we will make it the same as our Online Scene as set above.
- Game Scene
  - The scene to eventully be loaded into

## Planning

Now that we understand that Mirror is looking for let's at what we need to accomplish our character builder

- Main Menu Scene
  - Menu Controller
    - Host/Join
  - Extended NetworkRoomManager
- Lobby (aka Room aka Character Builder) Scene
  - Lobby Controller
    - Select and sync index via Room Player for their character model
    - Select and sync weapon index via Room Player for their weapon model
  - Exteneded NetworkRoomPlayer
  - PlayerSyncVO which extends NetworkBehaviour
- Game Scene
  - These will be were the actual game play occurs. Most things related to the game scene are outside the scope of this post.
  - A controller to handle the loading of your models when the model indexes change

## Set Up

With that all thunk through, the actual implementation is pretty straightforward

### Create Game Scene

Nothing special needs to happen in this scene

### Create Lobby Scene

The lobby scene is also our Online, Character Builder and Room scene.

#### LobbyController.cs

```csharp
// We fire off this trio of Commands (Commands are messages from client to server) 
// when we click our ready button
roomPlayer.CmdUpdateCharacterModelIndex(modelIndex);
roomPlayer.CmdUpdateWeaponModelIndex(weaponIndex);
roomPlayer.CmdChangeReadyState(true);
                
```

### Create Player Controller

Aattach to your Player Prefab.

#### PlayerController.cs

```csharp
// Within Start function we get a reference to our PlayerSyncVO which at this point has
// been filled with the data from RoomPlayer
PlayerSyncVO roomPlayer = GetComponent<PlayerSyncVO>();
characerModelController.LoadCharacterModel(roomPlayer.CharacterModelIndex);
hammerHead = await characerModelController.LoadWeaponModel(roomPlayer.WeaponModelIndex);
```

### Create Player Sync VO

Attach to Player Prefab.

#### PlayerSyncVO

```csharp
using Mirror;

public class PlayerSyncVO : NetworkBehaviour
{
    [SyncVar]
    public int CharacterModelIndex;
    [SyncVar]
    public int WeaponModelIndex;
    [SyncVar]
    public bool IsWhacker;
}
```

### Create Characrter Model Controller

This is what does the heavy lifting of conveting the synced indexes to prefabs. It's outside of the scope of this post as it don't include any networking code but personally I just use addressables to store my prefabs and in this class convert the indexes to paths names to load the proper prefab.

### Create MainMenu Scene

Create NetworkRoomManagerExt, add to an object in scene and accosicate all the above elements to it.

#### NetworkRoomManagerExt.cs

```csharp
using UnityEngine;
using Mirror;

public class NetworkRoomManagerExt : NetworkRoomManager
{
    public override bool OnRoomServerSceneLoadedForPlayer(NetworkConnectionToClient conn, GameObject roomPlayer, GameObject gamePlayer)
    {
        NetworkRoomPlayerExt serverRoomPlayer = roomPlayer.GetComponent<NetworkRoomPlayerExt>();
        PlayerSyncVO roomPlayerVO = gamePlayer.GetComponent<PlayerSyncVO>();
        if (serverRoomPlayer.isServer && serverRoomPlayer.isLocalPlayer)
        {
            roomPlayerVO.IsWhacker = true;
        }

        roomPlayerVO.CharacterModelIndex = serverRoomPlayer.CharacterModelIndex;
        roomPlayerVO.WeaponModelIndex = serverRoomPlayer.WeaponModelIndex;
        return base.OnRoomServerSceneLoadedForPlayer(conn, roomPlayer, gamePlayer);
    }
}
```

### Create UI

This is just two input field and a button and is connected to MainMenuController.
  
#### MainMenuController.cs

```csharp
public void CreateHost()
{
    NetworkManager.singleton.StartHost();
}

public void JoinHost()
{
    NetworkManager.singleton.networkAddress = hostURL.text;
    NetworkManager.singleton.StartClient();
}
```

## Conclusion

With only a little code we were able to pretty easily setup a sync multiplayer character builder. This is just a simple example but could easily be scaled up to handle more complex character creations.
