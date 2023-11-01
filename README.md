# ESP (Extra-Sensory Perception) Analysis
The purpose of this repository is to understand what ESP is, how most games prevent ESP cheaters and gather an understanding as to why it is not a solved problem. Hopefully while offering some solutions, or at the very least some insight into some potential solutions.

# What is ESP?
Extra-Sensory Perception cheats are a form of cheat most commonly seen in online PvP games, they are used to see various bits of information about objects in the game world, even if you cannot visibly see them. This grants players an unfair advantage as they may be able to see enemy players through walls, easily find resources in the game world and much more.

![25ILogM](https://github.com/ryanjpwatts/esp-analysis/assets/33863267/aaa5016b-b443-4944-9cee-4627dd9f934c)

Above is an example of an ESP cheat in-use in CSGO, showing enemies behind walls with various game data labelled such as HP and name.

# How does ESP work?
ESP, like most cheats, rely on data stored in the game processes memory to function, this data is typically networked to the client from the game server. Every object in a game (players, npcs, items, etc.) will have some form of structure, programmatically in the form of classes or structs usually, this will hold the data about the object in a specific order that cheat developers are able to figure out through reverse engineering techniques and calculate the offsets for. For instance, suppose we have this C++ snippet:
```
class Player {
public:
    float position_x;
    float position_y;
    float position_z;
    const char* name;
};
```
Here we have a very basic class with two types of data, floats and a char pointer. Floats are 4 bytes in size and char pointers are 8 bytes (on 64-bit architecture). With this in mind we can deduce that if we are able to identify a Player pointer in memory, we can dereference this and read the first 4 bytes and have the position_x, the next 4 bytes will be our position_y, the next 4 bytes will be our position_z and the next 8 bytes will be a pointer to the name. Hooray, we have our data to create our ESP.

However, this alone is not enough to create an ESP, to actually translate this data into something we can output on our screen we would need to convert the 3D world coordinates of our object to 2D screen coordinates, internally injected cheats will tend to leverage a games own world to screen function to accomplish this while external cheats will do the calculation themselves. 

Additionally, more advanced ESPs may take it a step further and read specific data on a players bones to produce a more accurate outline, or perhaps even leverage the games functions to do the ESP rendering for them on specific entities.
# How is it currently prevented and what more can be done?
To understand how some games combat ESP cheating let's breakdown the two primary aspects of anti-cheating; client-side and server-side.
## Client-sided
Client-sided anti-cheat measures are methods to prevent and identify cheating that run on the clients machine. 

Generally, these types of solutions will run in the kernel and look for modifications of the games code, prevent external processes from reading the games memory (as a result making dynamic analysis more difficult), prevent suspicious DLLs from being injected into the game process as well as a myriad of other things. Generally speaking, current client-sided anti-cheating solutions don't do anything to prevent ESP cheats in particular but rather combat methodology to create and run cheats.

I personally believe there is little room for client-sided measures to achieve much more than they already have with regards to ESP in particular, anti-cheating measures have toyed with the idea of [taking screenshots of players screens](https://ucp-anticheat.org/screens.html) in the past. The Windows API offers functionality such as [setting the window display affinity](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowdisplayaffinity) to circumvent these screenshotting measures but there is never any harm in an additional detection vector.
## Server-sided
On the other hand, server-sided anti-cheat measures are methods to prevent, mitigate and identify cheating that run on the game server (or more typically backend game services). 

These usually take the form of rule engines, machine learning models and other behavioural analysis systems. These solutions are great at identifying cheating after it has happened assuming you have a great dataset, well-calibrated models/rules and a scalable system. These types of systems can help prevent ESPs due to the behaviour associated with players using these types of cheats, for instance these players will be more likely to aim at enemies through walls, directly path from precious resource to precious resource or carelessly traverse through potentially dangerous areas with the knowledge there are no threats.

An additional form of server-sided anti-cheat is simply server-authorative features, with online games being the classic example of the client-server architecture a lot of cheating can be prevented with a more server-authorative design, which goes on to the solution...

## The solution (sort of)
As covered in the [How does ESP work?](https://github.com/ryanjpwatts/esp-analysis#how-does-esp-work) section, we know we need the objects position data in memory to create these ESP hacks to begin with so it begs the question, why are games not taking a more server-authorative stance on this and simply not sending positional data for objects if they cannot be seen by the player?

Firstly, it's of course worth saying, it is easier said than done but it is a feasible option that would outright prevent ESPs as we know them from functioning at all. Let's look at a demo for this I made using Unity with Mirror Networking -

https://github.com/ryanjpwatts/esp-analysis/assets/33863267/74cf5827-c412-4ef5-86f3-aaea66a56c17

```
public class ESPInterestManagement : InterestManagement
{
    // How far in unity measurements we want our players to be able to see other players from
    private float viewDistance = 100.0f;
    // The fov value to render within, cheaters can still modify their fov but players won't be rendered outside of it
    private float fieldOfView = 60.0f;

    public override bool OnCheckObserver(NetworkIdentity identity, NetworkConnectionToClient newObserver)
    {
        return ValidateObserver(identity, newObserver);
    }

    public override void OnRebuildObservers(NetworkIdentity identity, HashSet<NetworkConnectionToClient> newObservers)
    {
        foreach (NetworkConnectionToClient conn in NetworkServer.connections.Values)
        {
            // if authenticated and joined the world
            if (conn != null && conn.isAuthenticated && conn.identity != null)
            {
                if (ValidateObserver(identity, conn.identity.connectionToClient))
                {
                    newObservers.Add(conn);
                }
            }
        }
    }

    private bool ValidateDistance(NetworkIdentity identity, NetworkConnectionToClient newObserver)
    {
        // in "render" distance?
        return Vector3.Distance(newObserver.identity.transform.position, identity.transform.position) <= viewDistance;
    }

    private bool ValidateFoVLoS(NetworkIdentity identity, NetworkConnectionToClient newObserver)
    {
        // is the client facing the other client?
        Vector3 targetDir = identity.gameObject.transform.position - newObserver.identity.transform.position;
        return Vector3.Angle(targetDir, newObserver.identity.transform.forward) <= fieldOfView;
    }

    private bool ValidateBlockers(NetworkIdentity identity, NetworkConnectionToClient newObserver)
    {
        // is there anything obstructing our view?
        int blockMask = 1 << 6;
        List<Vector3> angles = new() { new Vector3(0, 0, 0), -identity.transform.right, identity.transform.right, identity.transform.up, -identity.transform.up };

        foreach (Vector3 angle in angles)
        {
            if (!Physics.Linecast(identity.transform.position + angle, newObserver.identity.transform.position, out _, blockMask)) return true;
        }

        return false;
    }

    private bool ValidateObserver(NetworkIdentity identity, NetworkConnectionToClient newObserver)
    {
        if (!ValidateDistance(identity, newObserver)) return false;
        if (!ValidateFoVLoS(identity, newObserver)) return false;

        return ValidateBlockers(identity, newObserver);
    }

    [ServerCallback]
    void Update()
    {
        RebuildAll();
    }
}
```
This simple solution has a dedicated server running and 2 clients connected to it, making use of Mirror Networkings interest management solution. The process before the server will send packets regarding a client goes as followed:
1) Is the client within the view distance of the other client? I've set this to 100 units by default.
2) Is the client facing our other client? This is checked through the fieldOfView variable and also helps prevent fov cheating.
3) Are there objects marked with a blocking layer blocking every possible line of sight from our client to the other? This is done with physics-based culling by sending out 5 Linecasts from one clients camera position to another, covering all angles that the player should be able to see from (straight, left, right, above, below).
  
Once all conditions are validated it will return true and the server will proceed with sending the packets to the client.

The two major drawbacks of this solution is functionality and efficiency. 

Anti-cheating measures should never affect legitimate players experiences and if this solution fails for instance, causing one player to see another player and the other cannot or two players are unable to see each other you may have some very unhappy players, and this would be unacceptable in any competitive gaming scenario. This could be solved with a bit of leniency i.e. ensuring packets are sent just before they may turn a corner, keeping in mind that could partially compromise the effectiveness of our method, this should be something configurable dependent on how harsh the game server intends to enforce it.

With this in mind, we would want our solution to be as accurate as possible and as a result it may be slightly more computationally expensive than a few simple Unity Linecasts. When we need to factor in that the server will be running these checks every frame with each clients position being validated against every other clients position, we are potentially creating an exponential time complexity problem. Despite this there are a few things to consider to further minimize the performance impact, these may include:
- Ensuring the logic to validate line of sight is not running unnecesary additional checks if line of sight has been established, like I have done in the example above.
- Applying game-specific logic to minimize server calculations (i.e. we can send packets containing data about teammates without major cheating concerns, regardless of line of sight).
- Disabling the feature in high player density areas. While this obviously works against what we are trying to solve, a computationally expensive operation could be exploited by bad actors to lag servers.
- Not running these checks every frame on the server. In a more casual gaming setting this may be an acceptable compromise and would be a huge boost in performance.

Functionality and efficiency aside, this of course is no silver bullet to ESPs, another component related to certain objects is audio. For example, should a player fire a weapon behind a wall or walk/run there will be a position stored in memory for the source of the audio which could be used to create a form of ESP, but this is substantially less beneficial than what they are currently capable of and should not be a reason to not pursue this as a method to combat ESPs.

# Performance
soon

# Closing thoughts
After spending an afternoon and evening thinking about and implementing this solution it is very interesting that games with few players in a single server instance are not all successfully expirementing and releasing similar solutions as this would not only prevent ESPs but also prevent certain types of aimbots as well (more specifically the ones associated with "rage hacking" that shoot players through walls). 

While researching this topic I watched gameplay of players using ESP hacks on CS2 and Valorant as I expected these two games to be the perfect candidates to put this into practice. And as I expected, it would appear Valorant may use something similar as I did frequently observe ESPs having players popping in and out with the occassional teleporting. Strangely CS2 does not seem to implement anything resembling what I have showcased but it is good to know that some developers are at least considering being more restrictive with the packets they send to clients about object positions.

If you are aware of any games actively using similar methods to prevent ESP cheating I would be very interested to hear about it and please do [DM me on Twitter](https://twitter.com/ryan_jpw).


# Resources / Credits
[A very cool showcase from Sneaky Kitty Game Dev on how a more server-authorative approach could look](https://www.youtube.com/watch?v=znifN3rytnY)

[An interesting article from Serafim Pinto at Anybrain that covers some details about the solution I tested](https://blog.anybrain.gg/client-side-vs-server-side-anti-cheat-6721d38eb347)
