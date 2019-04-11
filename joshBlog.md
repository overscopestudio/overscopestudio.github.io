# Josh's Blog
 
 <!-- Nav -->
<table style="border:0px none;" width="100%">
	<tr>
		<td width="25%"> <a href="index.html"> Home </a> </td>
		<td width="25%"> <a href="amsBlog.html">Ashleys Blog</a> </td>
		<td width="25%"> <a href="jakobBlog.html">Jakob's Blog</a> </td>
		<td width="25%"> <a href="joshBlog.html">Josh's Blog</a> </td>
	</tr>
</table>

<!-- Main Content -->

## Main Menu Navigation Map
##### Date: 11/04/2019

I'm Josh, the youngest of three programmers working on Jazz Odyssey. My main project within this game was creating a menu map that would act as a central point within the game and act as a transition between the levels.

<details><summary> The final outcome looked like this: </summary>
<p> <img src="./Images/Jonch/map.PNG"> </p> </details>

There are 5 buttons around the map, each in their own unique area, and the single player sprite in the center, as well as many invisible waypoints scattered around that create the paths.

When I started thinking about this problem, I thought about a couple of ways I could go about setting this all up. I ended up going with the idea I would need a sort of Tree of objects in the scene, which would act as waypoints for the player to navigate across.
I created very crude actors to begin with, and scattered them around the map. I then created the player which received a location from the buttons when pressed, and then navigated to that location.
<details><summary>The crude code worked something like this:</summary>
<p>

```
// BUTTON // 
when button clicked:
	get location of self,
	send location to player object.
	
	
// PLAYER OBJECT //
loop forever:
	wait until a location is received WHEN I'm not busy,
	set self to busy,
	travel to location that was received,
	set self to NOT busy,
```

</p></details>

I very quickly scrapped this, as it was not expandable whatsoever and was mainly a proof of concept for my team.

Now onto making this expandable, I had little idea as to what I could do.
I thought up a new solution, and it was by ditching the old button all together and creating two different types of buttons. I started with a base class that would have both the buttons derive from it, the base class contained any shared property that was between them, at the time this was just a boolean that stored whether the button could be clicked or not.
I then derived the two button types from the base, Head and Leaf, basing it on the Tree data structure. The leaves where relatively simple, they could store an object that was in front of them, and when clicked, would first check if it was allowed to be clicked before sending its location to the object in front of it. If the object was another leaf, the next leaf would add its location to an array, and send the array forwards. The process repeated until the array of locations would be received by the Head.
When the Head received the array, it sent it the the Player object and it moved across the map according to the array.
 












