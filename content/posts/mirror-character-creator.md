+++
title = 'Mirror-Networking Character Creator'
date = 2023-11-13T21:56:14-04:00
+++

I am working on a little multiplayer Unity game and decided to use [Mirror Networking](https://mirror-networking.com/) for handling the server and communication. In this post I'm going to outline how I use Mirror to build a simple Character Builder and syncing that between clients.

## Mirror Concept Breakdown

We're going to leverage the [NetworkRoomManger](https://mirror-networking.gitbook.io/docs/manual/components/network-room-manager) for our game. The NetworkRoomManager handles the flow of a player connecting or becoming the host, create a room, ensure players are ready, transitioning to the game scene, creating your character prefab and along the way offering hooks for customing the different steps.

As you can see, NetworkRoomManager does a lot for us and only asks for a few objects to be configured in return; An offline scene, an online, your transport, a player prefab, a room player prefab, a room scene and a game scene.

Example:

![NetworkRoomManagerExt Config](static/mirror-character-creator/NetworkRoomManagerExt_Example.png)

Let's break these requirements down

- Offline scene
  - This is the scene that will contain your Monobehaviour that has your NetworkRoomManagerExt.cs attached. This is the the scene to boot your server. Likely your menu.
- Online Scene
  - This is the scene that the player transitions to once they join/host a game.
- Transport
  - This is the most straightforward. Make a choice on your protocal and set it.
- Player Prefab
  - ![Whacker_Controller_VO](static/mirror-character-creator/Whacker_Controller_VO.png)
  - Note WhackerController and PlayerSyncVO - These are NetworkBehaviour and are capiable of syncing across the server
- Room Player Prefab
  - ![Whacker_Controller_VO](static/mirror-character-creator/RoomPlayerPrefab.png)
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

- Create Game Scene
  - Nothing special needs to happen in this scene
- Create Lobby Scene
  - The lobby scene is also our Online, Character Builder and Room scene.

_LobbyController.cs_

```csharp
using System.Collections.Generic;
using Mirror;
using TMPro;
using UnityEngine;

namespace asymMole.App
{
    public enum State
    {
        CharacterLockIn,
        WeaponLockIn,
        Ready
    }

    public class LobbyController : MonoBehaviour
    {
        public State state;
        private int modelIndex;
        private int weaponIndex;
        private int selectionIndex;
        private List<string> selectedModelList;
        [SerializeField]
        private TMP_Text buttonLabel;
        [SerializeField]
        private CharacerModelController characterModelController;

        void Start()
        {
            state = State.CharacterLockIn;
            selectedModelList = characterModelController.CharacterModels;
            UpdateModelForIndex(0);
        }

        public void OnPrev()
        {
            selectionIndex--;
            if (selectionIndex < 0)
            {
                selectionIndex = selectedModelList.Count - 1;
            }

            UpdateModelForIndex(selectionIndex);
        }

        public void OnNext()
        {
            selectionIndex++;
            if (selectionIndex > selectedModelList.Count - 1)
            {
                selectionIndex = 0;
            }

            UpdateModelForIndex(selectionIndex);
        }

        public void LockIn()
        {
            switch (state)
            {
                case State.CharacterLockIn:
                    selectionIndex = 0;
                    selectedModelList = characterModelController.WeaponModels;
                    state = State.WeaponLockIn;
                    break;
                case State.WeaponLockIn:
                    selectionIndex = 0;
                    if (NetworkClient.localPlayer == null)
                    {
                        NetworkClient.AddPlayer();

                    }
                    NetworkRoomPlayerExt roomPlayer = NetworkClient.localPlayer.GetComponent<NetworkRoomPlayerExt>();
                    // Here is where we update the sever with our selected values
                    roomPlayer.CmdUpdateCharacterModelIndex(modelIndex);
                    roomPlayer.CmdUpdateWeaponModelIndex(weaponIndex);
                    roomPlayer.CmdChangeReadyState(true);
                    break;
                case State.Ready:
                    state = State.CharacterLockIn;
                    break;
                default:
                    Debug.Log("Err");
                    break;
            }

            UpdateUIForState();
        }

        private void UpdateUIForState()
        {
            switch (state)
            {
                case State.CharacterLockIn:
                    buttonLabel.text = "Lock in Character";
                    break;
                case State.WeaponLockIn:
                    buttonLabel.text = "Lock in Weapon";
                    break;
                case State.Ready:
                    buttonLabel.text = "Readied";
                    break;
                default:
                    break;
            }
        }

        private void UpdateModelForIndex(int selectedIndex)
        {
            if (state == State.CharacterLockIn)
            {
                modelIndex = selectedIndex;
            }

            if (state == State.WeaponLockIn)
            {
                weaponIndex = selectedIndex;
            }

            characterModelController.LoadCharacterModel(modelIndex);
            characterModelController.LoadWeaponModel(weaponIndex);
        }
    }
}
```

-
  - Create Player Controller and attach to your Player Prefab
_PlayerController.cs_

```csharp
using Cinemachine;
using DG.Tweening;
using Mirror;
using StarterAssets;
using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.InputSystem.UI;

public class PlayerController : NetworkBehaviour
{
    [SerializeField]
    private ThirdPersonController thirdPersonController;
    private Transform hammerHead;
    [SerializeField]
    private CharacerModelController characerModelController;

    async void Start()
    {
        CinemachineVirtualCamera whackeeCam = GameObject.FindGameObjectsWithTag("whackeeCam")[0].GetComponent<CinemachineVirtualCamera>();
        CinemachineVirtualCamera whackerCam = GameObject.FindGameObjectsWithTag("whackerCam")[0].GetComponent<CinemachineVirtualCamera>();
        InputSystemUIInputModule inputModule = GameObject.FindGameObjectsWithTag("inputsystem")[0].GetComponent<InputSystemUIInputModule>();

        CinemachineVirtualCamera selectedCam;

        PlayerSyncVO roomPlayer = GetComponent<PlayerSyncVO>();
        characerModelController.LoadCharacterModel(roomPlayer.CharacterModelIndex);
        hammerHead = await characerModelController.LoadWeaponModel(roomPlayer.WeaponModelIndex);

        if (!roomPlayer.IsWhacker)
        {

            whackerCam.Priority = 10;
            whackeeCam.Priority = 1;
            selectedCam = whackerCam;
        }
        else
        {
            transform.localScale = new Vector3(3, 3, 3);
            whackerCam.Priority = 1;
            whackeeCam.Priority = 10;
            selectedCam = whackeeCam;
        }

        if (roomPlayer.isLocalPlayer)
        {
            selectedCam.LookAt = transform;
        }

        GetComponent<PlayerInput>().uiInputModule = inputModule;
        GetComponent<PlayerInput>().camera = selectedCam.GetComponent<Camera>();
    }

    void OnSelect()
    {
        Sequence mySequence = DOTween.Sequence();
        mySequence.Append(hammerHead.transform.DOLocalRotate(new Vector3(99, 0, 0), 0.25f))
          .PrependInterval(0.2f)
          .Append(hammerHead.transform.DOLocalRotate(new Vector3(0, 0, 0), 0.25f));
    }
}

```

-
  - Create Player Sync VO and also attach to Player Prefab

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

-
  - Create Characrter Model Controller. This is what does the heavy lifting of loading the synced indexes. You'll see below I am using addressables to store my prefabs just convert the indexs to paths names to load the proper prefab.

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public struct ActiveObject
{
    public ActiveObject(string name, GameObject model)
    {
        Name = name;
        Model = model;
    }
    public GameObject Model;
    public string Name;
}

public class CharacerModelController : MonoBehaviour
{
    [SerializeField]
    private Transform pivot;
    public List<string> CharacterModels
    {
        get
        {
            return new List<string>()
            {
                "Characters/crab.prefab",
                "Characters/hen.prefab",
                "Characters/penguin.prefab",
                "Characters/pig.prefab",
                "Characters/shark.prefab",
                "Characters/t-rex.prefab"
            };
        }
    }

    public List<string> WeaponModels
    {
        get
        {
            return new List<string>()
            {
                "Weapons/axe.prefab",
                "Weapons/hammer.prefab",
                "Weapons/pickaxe.prefab",
                "Weapons/shovel.prefab",
                "Weapons/sledgehammer.prefab",
            };
        }
    }

    private ActiveObject currentCharacter;
    private ActiveObject currentWeapon;

    public async void LoadCharacterModel(int index)
    {

        if (currentCharacter.Model != null)
        {
            currentCharacter.Model.transform.SetParent(null);
            Destroy(currentCharacter.Model);
            currentCharacter.Model = null;
        }


        GameObject userModelPrefab = await LoadAsset(CharacterModels[index]);
        GameObject model = Instantiate(userModelPrefab, transform);
        currentCharacter = new ActiveObject(CharacterModels[index], model);
        Transform handPosition = FindModelsHand(model);
        PositionWeaponPiviotToHand(pivot, handPosition);
    }

    public async Task<Transform> LoadWeaponModel(int index)
    {
        if (currentWeapon.Model != null)
        {
            currentWeapon.Model.transform.SetParent(null);
            Destroy(currentWeapon.Model);
            currentWeapon.Model = null;
        }

        GameObject userWeaponPrefab = await LoadAsset(WeaponModels[index]);
        GameObject model = Instantiate(userWeaponPrefab, pivot);
        currentWeapon = new ActiveObject(WeaponModels[index], model);
        return model.GetComponentInChildren<Transform>();
    }

    private void PositionWeaponPiviotToHand(Transform pivot, Transform handPosition)
    {
        pivot.transform.position = handPosition.position;
    }

    private Transform FindModelsHand(GameObject userModel)
    {
        return Utils.FindTagInChildren(gameObject, "character_hand");
    }

    private async Task<GameObject> LoadAsset(string model)
    {
        AsyncOperationHandle<GameObject> loadHandle = Addressables.LoadAssetAsync<GameObject>(model);
        await loadHandle.Task;

        if (loadHandle.Status != AsyncOperationStatus.Succeeded)
        {
            Debug.Log("Error loading model");
            return null;
        }

        return loadHandle.Task.Result;
    }
}
```

- Create MainMenu Scene
  - Create NetworkRoomManagerExt, add to an object in scene and accosicate all the above elements to it.

_NetworkRoomManagerExt.cs_

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

-
  - Create UI (two input field and a button) and connect to MainMenuController
  
_MainMenuController.cs_

```csharp
using Mirror;
using TMPro;
using UnityEngine;

namespace asymMole.App
{
    public class MainMenuController : MonoBehaviour
    {
        [SerializeField]
        private TMP_InputField hostURL;
    
        public void CreateHost()
        {
            NetworkManager.singleton.StartHost();
        }

        public void JoinHost()
        {
            NetworkManager.singleton.networkAddress = hostURL.text;
            NetworkManager.singleton.StartClient();
        }
    }
}
```

## Conclusion

With only a little code we were able to pretty easily setup a sync multiplayer character builder. This is just a simple example but could easily be scaled up to handle more complex character creations.
