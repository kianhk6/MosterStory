# Encounter Technical Documentation
This document explains the technical aspects of our encounter system, detailing how interactive narrative elements are managed dynamically in our Unity game.
#### Project Setup Details

- **Unity Version:** 6000.0.34F1  
- **Ink API Version:** 1.2.1
  
## Main UI Elements in Encounter

The main UI elements displayed during an encounter are generated dynamically. In this context, "dynamic" means that these elements are automatically created based on the current story progression, the choices made by the player, and preloaded game data.

---

### Attribute Cards Prefab

Each Attribute Card is derived from a custom asset data type. The art for the card is created dynamically based on several fields.

#### Creating an Attribute Card

1. **Navigation:**  
   Navigate to `resources/AttributeCards`

2. **Action:**  
   Right-click and select **Create > Attribute Cards**

3. **Field Setup:**  
   Fill out the following fields:

##### Main Attribute Ratings (Integers)
- **Cold**, **Volatile**, **Dark**, **Hot**

##### Card Descriptors
- **Title Text:**  
  The text describing the attribute card.
- **Custom Card Sprite:**  
  The primary custom image (sprite) for this Attribute Card.  
  *Note: This is not the entire attribute card art asset; it is just the main image displayed on the card.*

##### Propagated Field (Based on the Script)
- **tag Images:**  
  A list of images that display the card’s attributes.  
  *Note: The script checks if the `cold`, `volatile`, `dark`, and `hot` attributes are non-zero, then displays the image that matches the corresponding attribute and value.*

### Dialogue Choice Prefab

The Dialogue Choice Prefab is a dynamic UI component that displays interactive dialogue options during an encounter. It automatically adapts based on the story context and player interactions. The prefab includes the following fields:

1. **Main Text**  
   Displays the choice’s text that the player sees.

2. **Card Placeholder**  
   Acts as the drop zone for the player’s card selection. Its behavior depends on the type of choice:
   - **Regular Dialogue Choices:**  
     The placeholder shows two attribute tags. To select the choice, the player’s attribute card must have matching tags.
   - **Move Choices:**  
     The placeholder remains empty, allowing the player to drop any move card without needing to match any specific tags.

## Game Context Management

The story context management comes from Ink:

> **Ink is a specialized scripting language** designed for interactive storytelling. In our project, we use Ink to structure our narrative, allowing our story to branch into multiple paths based on player choices. Our narrative writer creates the story using Ink, which is then compiled into a JSON file. This JSON file is imported into our Unity game, where API calls read and process the data to dynamically display dialogue and choices on screen. This setup lets us manage complex branching storylines seamlessly, without manually coding each dialogue variation.

The main scripts that handle the game context—such as available choices and narrative progression—are as follows:

### Dialogue Factory

Dialogue Factory is our main script for managing all in-game dialogues and choices. It serves as the central hub that reads the narrative JSON (compiled from Ink by our narrative writer) and leverages Ink’s API to drive the story forward. Essentially, it bridges the gap between our interactive story data and the dynamic UI elements in Unity, ensuring that dialogue and branching choices update as the story unfolds.


**Initialization (Start):**  
When the game begins, Dialogue Factory loads the Ink narrative JSON file and creates an Ink Story object. This step initializes our interactive narrative, setting the stage for dynamic dialogue and choice progression. Immediately after loading, the script calls `RefreshView()` to display the opening dialogue and choices.

**RefreshView Method:**  
`RefreshView()` is the core method that updates our UI as the story progresses. It dynamically refreshes the dialogue and available choices based on the current state of the Ink story. Its main tasks include:

1. **Choice Cleanup:**  
   Before adding new choices, the method removes any previously generated options that weren’t selected—keeping only the history of chosen dialogues. This ensures the UI remains uncluttered and accurately reflects the story’s progression.

2. **Narrative Text Propagation:**  
   The script repeatedly calls the Ink API’s `Continue()` method to retrieve the latest narrative text (typically spoken by NPCs). Each piece of text is appended to the dialogue history, forming a continuous narrative log that the player can follow.

3. **Choice Processing and Parsing:**  
   The method then checks for available choices provided by Ink. For each choice:
   - It parses the text to extract:
     - **Choice Type:**  
       - Dialogue Choice (`@`): Regular dialogue options requiring matching attribute tags.
       - Move Choice (`$`): Options related to player moves, identified by a specific keyword.
       - Outcome Choice (`&`): Options that represent the results of move actions, determined automatically.
     - **Tags or Keyword:**  
       For dialogue choices, a list of comma-separated attribute tags is extracted; for move and outcome choices, a single keyword is identified.
     - **Main Text:**  
       The portion of the text before any marker is used as the display text for the choice.

4. **UI Propagation:**  
   Based on the parsed data:
   - **Regular Dialogue Choices:**  
     A dialogue choice prefab is instantiated and updated with the main text and corresponding attribute tags. This allows the player to select the option only if their attribute cards match the displayed tags.
   - **Move Choices:**  
     A move card prefab is used. The parsed keyword directs which move card to update, ensuring that the appropriate move text (extracted from Ink) is displayed.
   - **Move Outcomes:**  
     Instead of a UI element, the move outcome is determined automatically based on the score of the attribute cards dropped during a move.

### Player Choice Activation

#### Regular Dialogue Choices

When a player selects a regular dialogue choice by dropping their attribute card into the designated play area, the system triggers the **Output()** function. This function performs the following steps:

- **Sound Feedback:**  
  It plays a sound corresponding to the card drop, providing immediate auditory feedback.

- **Story Branching:**  
  The function communicates with the Ink narrative by calling `ChooseChoiceIndex()`, passing in the index of the selected choice. This informs Ink which narrative branch to follow.

- **UI Update:**  
  Finally, **Output()** calls `RefreshView()` to update the dialogue history and display the new narrative text and choices. This ensures that the game context and UI are seamlessly synchronized with the player's selection.

#### Move Choices

The process for move choices involves a two-step interaction:

1. **Move Card Selection:**  
   - The player begins by selecting a move card from the available options.  
   - Based on the move card information propagated from Ink—specifically, the keyword and main text—the appropriate move description is displayed on the dashboard, guiding the player on the move’s details.

2. **Attribute Card Evaluation and Outcome Determination:**  
   - After selecting a move card, the player places attribute cards (and any modifiers) into the play area.
   - Once the continue button is pressed, the system calls functions within Dialogue Factory that engage the **MoveCalc** script.
   - **MoveCalc** calculates a score based on the attribute values and modifiers from the played cards.  
     - **Low Score:** Indicates a failure, triggering a specific sound effect.
     - **Medium Score:** Represents a partial success, with its own sound feedback.
     - **High Score:** Means the move succeeded, and the corresponding success sound is played.
   - Depending on the calculated score, **MoveCalc** calls `outputMove()` with a parameter reflecting the outcome (e.g., 1 for success, 2 for partial success, 3 for failure). This call then updates the narrative branch in Ink to reflect the move’s outcome.

## Card mechanics

The card mechanism is a core interactive feature that allows players to select dialogue options or moves by manipulating cards. It is built using two main components: the **CardMovement** script and the **CardDropZone** script. Together, they manage the card’s visual states, detect when a card is dropped onto a valid area, and trigger the corresponding game logic.

#### CardMovement Script

This script is attached to every attribute card and governs its behavior through several states:

- **State 0 – Default (Idle):**  
  When the card is in the player’s hand, it remains in an idle state without any special effects.

- **State 1 – Hover:**  
  When the mouse pointer hovers over the card, the script triggers a hover state. The card scales up and a glow effect is activated, indicating that it is interactive.

- **State 2 – Drag:**  
  When the player clicks and drags the card, it enters the drag state. In this state, the card follows the mouse pointer, updating its position in real time.

- **State 3 – Play:**  
  When the card is released (dropped), the script checks if it is close enough to a valid drop zone. If it is, the card transitions to the play state, is snapped to the drop zone’s position, and its parent is updated accordingly.

#### CardDropZone Script

This script is attached to drop zone objects, which are target areas where cards can be dropped. Each drop zone:

- Has a defined detection radius that determines whether a card is close enough to be considered “dropped.”
- Stores an index corresponding to a specific dialogue choice.  
- May provide compatibility checks (via its parent components) to verify if the card’s attributes match the expected criteria.

#### Determining the Activated Card

When the card is dropped:
- **Proximity Detection:**  
  The **CardMovement** script continuously checks the distance between the card and available drop zones. If the card comes within the drop zone’s detection radius, the system recognizes it as a potential activation.
  
- **Compatibility Check:**  
  For dialogue choices, the drop zone (or its parent display component) checks if the card’s attributes match the required tags. Only compatible cards can activate a choice.
  
- **State Transition & Trigger:**  
  Once the card meets the drop zone’s criteria, it transitions to state 3 (play state). At this point, the card is anchored to the drop zone, and a sound effect (e.g., “Play_PutInSlot”) is played.  
  - For regular dialogue choices, the system calls the `DialogueFactory.output()` function with the appropriate choice index to advance the story.
  - For move choices, additional processing occurs (via the MoveCalc script) to evaluate the move outcome before calling `outputMove()`.

This structured approach ensures that the card which is successfully dropped into a valid area and passes compatibility checks is the one activated—triggering the associated narrative branch or move outcome.

## Hand and Deck Management

Our game uses a robust card management system to handle the player's attribute cards. This system is composed of two main components:

- **Deck Manager:** Manages loading, organizing, and drawing cards.
- **Hand Manager:** Manages the player’s hand, moving cards between deck, hand, and discard piles, and updating the UI.


### Deck Manager

The Deck Manager is responsible for the initial setup and continuous replenishment of the player’s cards. Its key functions include:

- **Loading Card Assets:**  
  At startup, the Deck Manager automatically loads all attribute card assets from the Resources folder (specifically from the "AttributeCards" directory) and stores them in an internal list.

- **Drawing Cards:**  
  It populates the player’s deck by drawing a predetermined number of cards (e.g., 12 cards) and passing them to the Hand Manager. This drawing mechanism is cyclic—using modulo arithmetic to loop back to the start—ensuring that there is always a steady supply of cards available.


### Hand Manager

The Hand Manager oversees the state and visual representation of the player's cards. It maintains three distinct collections:

- **Cards in Hand:**  
  These are the cards currently available for the player to use. The Hand Manager updates their positions in a “fan” layout, which makes the hand visually appealing and easy to interact with.

- **Cards in Deck:**  
  Cards waiting to be drawn are held here. When the hand has space, cards are transferred from the deck to the hand.

- **Cards in Discard:**  
  Once a card is played or no longer needed, it is moved to the discard pile. When the deck runs out, the Hand Manager shuffles the discard pile and transfers those cards back into the deck, ensuring continuous gameplay.

**Key Functions:**

- **Update Hand Visuals:**  
  The Hand Manager dynamically arranges the cards in the player's hand by adjusting their positions, rotations, and spacing. This “fan” effect visually distinguishes the hand from the deck and discard piles.

- **Card Movement:**  
  Cards transition between the deck, hand, and discard piles through dedicated functions. For example:
  - **AddCardToHand():** Transfers a card from the deck to the hand, ensuring the hand does not exceed a set limit.
  - **AddCardToDiscard():** Moves a played card into the discard pile while triggering sound effects for feedback.
  - **ReAddCardsToDeck():** When the deck is empty, it shuffles the discard pile and replenishes the deck, ensuring the player always has cards available.

- **UI Updates:**  
  As cards are added or removed, the Hand Manager updates their positions on screen, keeping the display current and responsive.

### Integration

- The Deck Manager continuously supplies new cards.
- The Hand Manager ensures that cards are displayed correctly, handles their movement between states, and provides visual and auditory feedback for player interactions.



