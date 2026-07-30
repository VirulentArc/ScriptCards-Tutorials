# Learning ScriptCards by Building a Dynamic Character Ability

By the end, you will have written a ScriptCard that:

- finds a character named `TestSheet`;
- builds the text of a Roll20 macro one piece at a time;
- preserves attribute calls such as `@{strength}` instead of resolving them early;
- reads every row in a repeating section;
- adds a line for each qualifying row;
- and creates a Character Ability named `NewAbility` containing the finished macro.

The final example uses the D&D 5E 2014 inventory section, but the method is reusable with other character sheets.

---

## What You Need Before Starting

You need:

1. ScriptCards installed in the game's Mod sandbox.
2. A character named exactly `TestSheet`.
3. Permission to create and edit Character Abilities.
4. For the repeating-section lessons, several inventory rows on `TestSheet`.

For the D&D 5E 2014 sheet, add a few inventory items and mark at least one as equipped.

Do not begin by copying the final script. Work through the smaller scripts in order. Each lesson introduces one new part of the language.

---

# 1. How ScriptCards Reads a Script

Every ScriptCard begins and ends with this wrapper:

```scard
!script {{
  SCRIPT STATEMENTS GO HERE
}}
```

Inside the wrapper, most statements have the same basic shape:

```text
--TAG|CONTENT
```

The pieces are:

| Part | Purpose |
|---|---|
| `--` | Tells ScriptCards that a new statement begins |
| `TAG` | Tells ScriptCards what kind of statement this is |
| <code>&#124;</code> | Separates the tag from the content |
| `CONTENT` | The information used by that statement |

The first character of the tag usually identifies the command.

For example:

```scard
--#title|My First ScriptCard
```

means:

```text
--        Begin a ScriptCards statement
#title    Set the title parameter
|         Separate the tag from its content
My First ScriptCard
          The value assigned to the title
```

ScriptCards normally executes statements from top to bottom.

## Your First Complete ScriptCard

Create a Roll20 macro and paste this into it:

```scard
!script {{
  --/|This is a comment. ScriptCards ignores it.
  --#title|My First ScriptCard
  --+Result|The script ran successfully.
}}
```

Run the macro.

You should see a card titled **My First ScriptCard** with a row labelled **Result**.

## What Each Statement Does

```scard
--/|This is a comment. ScriptCards ignores it.
```

The `/` command creates a comment. Comments are useful for explaining why a section exists.

```scard
--#title|My First ScriptCard
```

The `#` command changes a ScriptCards setting. Here it changes the card title.

```scard
--+Result|The script ran successfully.
```

The `+` command adds a visible row to the card.

The tag is `+Result`, so `Result` becomes the left side of the row. The content becomes the right side.

## Try It Yourself

Before continuing:

1. Change the title.
2. Change the output text.
3. Add a second `--+` row.

For example:

```scard
--+Second Row|I added this myself.
```

The goal is to become comfortable with the statement structure, not merely to make the example run.

---

# 2. Storing Information in String Variables

Our finished script will need to remember several pieces of text:

- the character name;
- the Ability name;
- and the macro text we are building.

ScriptCards string variables are created with `--&`.

```scard
--&CharacterName|TestSheet
```

This creates a string variable named `CharacterName` and stores `TestSheet` in it.

To read the variable later, use:

```text
[&CharacterName]
```

## Build a Card with Variables

```scard
!script {{
  --#title|String Variables

  --&CharacterName|TestSheet
  --&AbilityName|NewAbility
  --&AbilityText|This is a test

  --+Character|[&CharacterName]
  --+Ability|[&AbilityName]
  --+Contents|[&AbilityText]
}}
```

ScriptCards replaces each `[&VariableName]` reference with the value currently stored in that variable.

## Setting and Appending Are Different

This statement replaces the variable's value:

```scard
--&AbilityText|This is
```

This statement appends text because the content begins with `+`:

```scard
--&AbilityText|+ a test
```

Together, they produce:

```text
This is a test
```

Try it:

```scard
!script {{
  --#title|Building a String

  --&AbilityText|This
  --&AbilityText|+ is
  --&AbilityText|+ a
  --&AbilityText|+ test

  --+Finished Text|[&AbilityText]
}}
```

This append pattern is important. Later, `AbilityText` will begin as a roll template and gain one additional field for every repeating row.

## A Useful Habit: Inspect Before Acting

When building text, display it before writing it anywhere:

```scard
--+Current Text|[&AbilityText]
```

A builder script is much easier to debug when you can see the string it has assembled.

---

# 3. Plan the Script Before Writing Commands

The first requested task is:

> On `TestSheet`, create an Ability named `NewAbility` whose action is `This is a test`.

Before choosing ScriptCards commands, rewrite that requirement as small operations:

```text
1. Store the character name.
2. Store the Ability name.
3. Store the Ability action.
4. Obtain the character's Roll20 ID.
5. Create the Ability on that character.
6. Report what happened.
```

This plain-language outline is the design of the script.

Only after the steps are clear should we translate each one into ScriptCards syntax.

---

# 4. Create the First Character Ability

Here is the smallest useful version:

```scard
!script {{
  --#title|Ability Builder

  --/|Configuration
  --&CharacterName|TestSheet
  --&AbilityName|NewAbility-Step1
  --&AbilityText|This is a test

  --/|Roll20 resolves this named character reference.
  --&CharacterID|@{TestSheet|character_id}

  --/|Create the Character Ability.
  --!ob:NewAbilityID:[&CharacterID]:[&AbilityName]:n|[&AbilityText]

  --/|Report the result.
  --+Created|[&AbilityName] was created on [&CharacterName].
  --+Object ID|[&NewAbilityID]
}}
```

Run it, then open `TestSheet` and inspect **Attributes & Abilities**.

You should find:

```text
Name: NewAbility-Step1
Action: This is a test
```

## Understanding the Character Lookup

```scard
--&CharacterID|@{TestSheet|character_id}
```

Roll20 processes `@{TestSheet|character_id}` before ScriptCards begins executing. The result is the internal ID of `TestSheet`.

ScriptCards then stores that ID in `[&CharacterID]`.

This is intentional. At this stage, we want the character reference resolved immediately.

### Using a Selected Token Instead

When you want the same builder to operate on whichever character token is selected, replace:

```scard
--&CharacterID|@{TestSheet|character_id}
```

with:

```scard
--&CharacterID|@{selected|character_id}
```

The script must be run with a token selected, and that token must represent a character. Roll20 resolves the selected token's character ID before ScriptCards begins executing.

The named-character form is useful when the builder should always target one specific character. The selected-token form is more reusable when the same script should work for different characters.

## Understanding `--!ob`

The Ability-creation statement is:

```scard
--!ob:NewAbilityID:[&CharacterID]:[&AbilityName]:n|[&AbilityText]
```

Its structure is:

```text
--!ob:RETURN_VARIABLE:CHARACTER_ID:ABILITY_NAME:TOKEN_ACTION|ABILITY_ACTION
```

For this script:

| Part | Value |
|---|---|
| Return variable | `NewAbilityID` |
| Character ID | `[&CharacterID]` |
| Ability name | `[&AbilityName]` |
| Token action | `n` |
| Ability action | `[&AbilityText]` |

ScriptCards stores the new Ability object's ID in:

```text
[&NewAbilityID]
```

Use `y` instead of `n` when the created Ability should be shown as a token action.

## Important: Creation Is Not Replacement

`--!ob` creates a new Ability. It does not search for and replace another Ability with the same name.

While learning, either:

- use a different Ability name for each experiment; or
- delete the previous test Ability before rerunning the same creation script.

The final script will check whether `NewAbility` already exists and stop rather than create a duplicate.

## Exercise

Change these three variables:

```scard
--&CharacterName|TestSheet
--&AbilityName|NewAbility-Step1
--&AbilityText|This is a test
```

Then adjust the character lookup when using a different character name:

```scard
--&CharacterID|@{Different Character Name|character_id}
```

Notice that the variables make most of the script reusable, but the Roll20 character call must also refer to the intended character.

---

# 5. Build the Ability Action Instead of Writing It All at Once

The previous script stored a completed action:

```scard
--&AbilityText|This is a test
```

A dynamic builder works differently. It starts with one piece and appends more pieces as it discovers them.

Try this version:

```scard
!script {{
  --#title|Constructing an Ability Action

  --&AbilityText|This
  --+Step 1|[&AbilityText]

  --&AbilityText|+ is
  --+Step 2|[&AbilityText]

  --&AbilityText|+ a test
  --+Step 3|[&AbilityText]
}}
```

The variable changes as the script executes:

```text
Step 1: This
Step 2: This is
Step 3: This is a test
```

That is the central pattern of this tutorial:

```text
Start the macro text.
Read one piece of sheet data.
Append the text produced by that data.
Repeat.
Create the Ability from the completed string.
```

Before adding repeating rows, we first need to solve a more subtle problem: preserving Roll20 attribute calls inside the generated Ability.

---

# 6. Understand When Roll20 and ScriptCards Expand Text

Suppose the generated Ability must contain:

```text
@{strength}
```

The important word is **contain**. We do not want the builder to read Strength. We want it to write the characters `@{strength}` into the new Ability so that the new Ability reads Strength later.

There are two different processing stages.

## Stage 1: Roll20 Processes the Command

Before ScriptCards receives the macro, Roll20 sees and resolves expressions such as:

```text
@{TestSheet|character_id}
@{TestSheet|strength}
?{A Roll Query}
```

## Stage 2: ScriptCards Executes the Result

After that, ScriptCards processes its own references:

```text
[&StringVariable]
[*S:attribute]
[*R:repeating_field]
```

This means the following builder line is wrong for our purpose:

```scard
--&AbilityText|@{TestSheet|strength}
```

Roll20 will probably replace that call with the current Strength value before ScriptCards receives it. The generated Ability would contain something like:

```text
16
```

instead of:

```text
@{strength}
```

## Construct Reserved Syntax After ScriptCards Starts

Prevent early resolution by ensuring the original Roll20 command never contains the complete sequence `@{`.

Build it from two assignments:

```scard
--&AtOpen|@
--&AtOpen|+{
```

After the second line, `[&AtOpen]` contains:

```text
@{
```

Also store a closing brace:

```scard
--&CloseBrace|}
```

Now ScriptCards can assemble:

```scard
--&AbilityText|[&AtOpen]strength[&CloseBrace]
```

The finished value is:

```text
@{strength}
```

But that sequence did not exist until ScriptCards was already running, so Roll20 could not resolve it early.

## Test the Idea Without Creating an Ability

```scard
!script {{
  --#title|Deferred Attribute Test

  --&AtOpen|@
  --&AtOpen|+{
  --&CloseBrace|}

  --&AbilityText|[&AtOpen]strength[&CloseBrace]

  --+Generated Text|[&AbilityText]
}}
```

The output should show:

```text
@{strength}
```

This is an important checkpoint. Do not move on until the output is the literal attribute call rather than a number.

---

# 7. Build a Roll Template from Safe Pieces

The requested next step is to write a template containing attributes that remain unresolved.

We want the generated Ability to contain:

```text
&{template:default} {{name=Attribute Test}} {{Strength=@{strength}}}
```

That text contains several sequences with special meaning to Roll20:

```text
&{
@{
{{
}}
```

Build each opening or closing sequence after ScriptCards starts.

```scard
--/|Creates @{
--&AtOpen|@
--&AtOpen|+{

--/|Creates &{template:
--&TemplateOpen|&
--&TemplateOpen|+{template:

--/|Creates {{
--&FieldOpen|{
--&FieldOpen|+{

--/|Creates }}
--&FieldClose|}
--&FieldClose|+}

--/|Creates }
--&CloseBrace|}
```

Now build the macro in readable stages:

```scard
--&AbilityText|[&TemplateOpen]default[&CloseBrace]
--&AbilityText|+ [&FieldOpen]name=Attribute Test[&FieldClose]
--&AbilityText|+ [&FieldOpen]Strength=[&AtOpen]strength[&CloseBrace][&FieldClose]
```

## Complete Inspection Script

```scard
!script {{
  --#title|Roll Template Builder

  --/|Construct reserved Roll20 syntax.
  --&AtOpen|@
  --&AtOpen|+{

  --&TemplateOpen|&
  --&TemplateOpen|+{template:

  --&FieldOpen|{
  --&FieldOpen|+{

  --&FieldClose|}
  --&FieldClose|+}

  --&CloseBrace|}

  --/|Build the Ability action one field at a time.
  --&AbilityText|[&TemplateOpen]default[&CloseBrace]
  --&AbilityText|+ [&FieldOpen]name=Attribute Test[&FieldClose]
  --&AbilityText|+ [&FieldOpen]Strength=[&AtOpen]strength[&CloseBrace][&FieldClose]

  --/|Inspect the result. Do not create anything yet.
  --+Generated Action|[&AbilityText]
}}
```

The finished string should be:

```text
&{template:default} {{name=Attribute Test}} {{Strength=@{strength}}}
```

## Why Build the String in Several Statements?

This is longer than typing the finished macro directly, but each line has one job:

1. create the template call;
2. append the title field;
3. append the Strength field.

That structure becomes valuable when later fields are generated by a loop.

## Exercise

Add a Dexterity field by appending one more statement:

```scard
--&AbilityText|+ [&FieldOpen]Dexterity=[&AtOpen]dexterity[&CloseBrace][&FieldClose]
```

Do not copy it without reading it. Identify:

- the field label;
- the opening of the deferred attribute call;
- the attribute name;
- and the two different closing sequences.

---

# 8. Create an Ability from the Generated Template

Once the inspection output is correct, use the completed string as the Ability action.

```scard
!script {{
  --#title|Template Ability Builder

  --/|Configuration
  --&CharacterName|TestSheet
  --&AbilityName|NewAbility-Step2
  --&CharacterID|@{TestSheet|character_id}

  --/|Construct reserved Roll20 syntax.
  --&AtOpen|@
  --&AtOpen|+{

  --&TemplateOpen|&
  --&TemplateOpen|+{template:

  --&FieldOpen|{
  --&FieldOpen|+{

  --&FieldClose|}
  --&FieldClose|+}

  --&CloseBrace|}

  --/|Build the action.
  --&AbilityText|[&TemplateOpen]default[&CloseBrace]
  --&AbilityText|+ [&FieldOpen]name=Attribute Test[&FieldClose]
  --&AbilityText|+ [&FieldOpen]Strength=[&AtOpen]strength[&CloseBrace][&FieldClose]
  --&AbilityText|+ [&FieldOpen]Dexterity=[&AtOpen]dexterity[&CloseBrace][&FieldClose]

  --/|Inspect immediately before writing.
  --+Generated Action|[&AbilityText]

  --/|Create the Ability.
  --!ob:NewAbilityID:[&CharacterID]:[&AbilityName]:n|[&AbilityText]

  --+Created|[&AbilityName] was created on [&CharacterName].
}}
```

Open the created Ability and confirm that it contains attribute calls, not the current attribute values.

Then run the created Ability. At that later point, Roll20 resolves `@{strength}` and `@{dexterity}` in the context of `TestSheet`.

The builder and the generated Ability perform different jobs:

| Script | Job |
|---|---|
| Builder ScriptCard | Writes the macro text |
| Generated Character Ability | Executes that macro text later |

Keeping those roles separate is essential.

---

# 9. Learn Repeating Sections Before Combining Them with the Builder

Do not begin by generating macro fields. First learn how ScriptCards walks through repeating rows.

For the D&D 5E 2014 inventory section, the relevant names are:

```text
Section:   repeating_inventory
Item name: itemname
Count:     itemcount
Equipped:  equipped
```

Other sheets use different section and field names.

## Load the First Row

```scard
!script {{
  --#title|First Inventory Row

  --&CharacterID|@{TestSheet|character_id}

  --Rfirst|[&CharacterID];repeating_inventory

  --+Item Name|[*R:itemname]
  --+Item Count|[*R:itemcount]
}}
```

`--Rfirst` loads the first row of the named repeating section.

After a row is loaded, `[*R:fieldname]` reads a field from the **currently loaded row**.

This is stateful. `[*R:itemname]` does not search every row. It reads `itemname` from whichever row ScriptCards currently has loaded.

## What Happens When There Is No Row?

When no repeating row is loaded, a repeating reference returns:

```text
NoRepeatingAttributeLoaded
```

A loop can test for that exact value to know when it has reached the end.

---

# 10. Walk Through Every Repeating Row

To process all rows, the script needs four control-flow tools:

| Command | Purpose |
|---|---|
| `--Rfirst` | Load the first row |
| `--Rnext` | Load the next row |
| `--:Label` | Mark a location in the script |
| `--^Label` | Jump to that location |
| `--?condition|Label` | Jump only when a condition is true |

Here is a complete inventory reader:

```scard
!script {{
  --#title|Inventory Reader

  --&CharacterID|@{TestSheet|character_id}

  --/|Load the first row.
  --Rfirst|[&CharacterID];repeating_inventory

  --/|The loop returns here after each Rnext.
  --:ReadInventoryRow|

  --/|Read a value from the current row.
  --&ItemName|[*R:itemname]

  --/|No loaded row means the loop is finished.
  --?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryDone

  --/|Use the current row.
  --+Item|[&ItemName]
  --+Count|[*R:itemcount]

  --/|Advance the repeating-row state.
  --Rnext|

  --/|Return to the beginning of the loop.
  --^ReadInventoryRow|

  --/|Execution arrives here after the end condition is true.
  --:InventoryDone|
  --+Finished|There are no more inventory rows.
}}
```

## Trace the Execution

Suppose the character has three inventory rows.

The script behaves like this:

```text
Rfirst loads row 1.
ReadInventoryRow reads and displays row 1.
Rnext loads row 2.
The branch returns to ReadInventoryRow.

ReadInventoryRow reads and displays row 2.
Rnext loads row 3.
The branch returns to ReadInventoryRow.

ReadInventoryRow reads and displays row 3.
Rnext finds no row and clears the repeating-row state.
The branch returns to ReadInventoryRow.

ItemName becomes NoRepeatingAttributeLoaded.
The conditional jumps to InventoryDone.
```

The loop does not know in advance that there are three rows. It keeps advancing until the next row does not exist.

## Why the End Test Comes Before Using the Row

This ordering is deliberate:

```scard
--&ItemName|[*R:itemname]
--?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryDone
--+Item|[&ItemName]
```

The script verifies that a row exists before treating the value as real sheet data.

## Exercise

Add another visible field from the repeating row, such as:

```scard
--+Weight|[*R:itemweight]
```

When adapting to another sheet, replace `itemweight` with a field that actually exists on that sheet.

---

# 11. Current Repeating Values Versus Repeating Attribute Names

This distinction is the key to generating a macro that remains dynamic.

## Read the Value Now

```text
[*R:itemcount]
```

This reads the current value from the loaded row.

It might return:

```text
3
```

If that value is appended to the generated Ability, the Ability permanently contains `3`.

## Read the Full Attribute Name

```text
[*R>itemcount]
```

The `>` form returns the complete repeating attribute name instead of its value.

It might return:

```text
repeating_inventory_-AbCdEf123_itemcount
```

That is still only the attribute name. It does not include `@{` or `}`.

Construct the deferred call:

```scard
--&DeferredCount|[&AtOpen][*R>itemcount][&CloseBrace]
```

The result is:

```text
@{repeating_inventory_-AbCdEf123_itemcount}
```

When the generated Ability runs later, Roll20 reads the current value from that exact repeating row.

## Probe Both Forms Side by Side

```scard
!script {{
  --#title|Repeating Reference Test

  --&CharacterID|@{TestSheet|character_id}

  --&AtOpen|@
  --&AtOpen|+{
  --&CloseBrace|}

  --Rfirst|[&CharacterID];repeating_inventory

  --+Value Right Now|[*R:itemcount]
  --+Full Attribute Name|[*R>itemcount]

  --&DeferredCount|[&AtOpen][*R>itemcount][&CloseBrace]
  --+Generated Attribute Call|[&DeferredCount]
}}
```

Do not continue until you understand the difference:

```text
[*R:itemcount]   asks for the value.
[*R>itemcount]   asks for the name needed to build a later reference.
```

---

# 12. Use the Loop to Build Macro Fields

We can now combine two patterns:

## String accumulator

```text
Start AbilityText.
Append more text to AbilityText.
```

## Repeating-row iterator

```text
Load the first row.
Use the current row.
Load the next row.
Repeat until no row exists.
```

The following script builds a roll template but does not yet create an Ability:

```scard
!script {{
  --#title|Repeating Template Builder

  --&CharacterID|@{TestSheet|character_id}

  --/|Construct reserved Roll20 syntax.
  --&AtOpen|@
  --&AtOpen|+{

  --&TemplateOpen|&
  --&TemplateOpen|+{template:

  --&FieldOpen|{
  --&FieldOpen|+{

  --&FieldClose|}
  --&FieldClose|+}

  --&CloseBrace|}

  --/|Start the macro text before entering the loop.
  --&AbilityText|[&TemplateOpen]default[&CloseBrace]
  --&AbilityText|+ [&FieldOpen]name=Inventory[&FieldClose]

  --/|Load the first inventory row.
  --Rfirst|[&CharacterID];repeating_inventory

  --:BuildInventoryField|

  --&ItemName|[*R:itemname]
  --?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryFieldsDone

  --/|Append one template field from the current row.
  --&AbilityText|+<br>[&FieldOpen][&ItemName]=[&AtOpen][*R>itemcount][&CloseBrace][&FieldClose]

  --Rnext|
  --^BuildInventoryField|

  --:InventoryFieldsDone|

  --/|Turn the temporary <br> separators into actual line feeds.
  --~AbilityText|string;brtolinefeeds;[&AbilityText]

  --/|Inspect the completed action.
  --+Generated Action|[&AbilityText]
}}
```

## Follow the Data Through One Row

Suppose the current row contains:

```text
itemname  = Potion of Healing
itemcount = 2
row ID    = -ExampleRowID
```

During the loop:

```text
[&ItemName]
```

becomes:

```text
Potion of Healing
```

and:

```text
[*R>itemcount]
```

becomes:

```text
repeating_inventory_-ExampleRowID_itemcount
```

The append statement produces:

```text
{{Potion of Healing=@{repeating_inventory_-ExampleRowID_itemcount}}}
```

Then `--Rnext` loads the next row and the same statement produces the next field.

## Why Use `<br>` Temporarily?

A string variable can safely accumulate a visible marker such as `<br>`:

```scard
--&AbilityText|+<br>MORE TEXT
```

After the loop, this function replaces those markers with actual line feeds:

```scard
--~AbilityText|string;brtolinefeeds;[&AbilityText]
```

This keeps the builder readable while creating a multi-line Ability action.

---

# 13. Include or Exclude Rows with a Conditional Block

The final requested behaviour is:

> Include a generated line only when another attribute has the correct value.

This is not the same as ending the loop.

There are now two questions for each iteration:

1. **Does a row exist?**
2. **Should this existing row be included?**

The first question controls the loop. The second controls only the append operation.

## Inspect the Controlling Field First

Before writing a condition, see what the sheet actually returns:

```scard
--+Equipped Value|[*R:equipped]
```

Checkboxes may be stored as `1`, `on`, `true`, or something sheet-specific.

The following example accepts either `1` or `on`.

## Conditional Block

```scard
--&Equipped|[*R:equipped]

--?"[&Equipped]" -eq "1" -or "[&Equipped]" -eq "on"|[
  --&AbilityText|+<br>[&FieldOpen][&ItemName]=[&AtOpen][*R>itemcount][&CloseBrace][&FieldClose]
--]|
```

The `[` begins a block of statements that run only when the condition is true.

The matching line closes the block:

```scard
--]|
```

## The Complete Loop with Both Decisions

```scard
--Rfirst|[&CharacterID];repeating_inventory

--:BuildInventoryField|

--&ItemName|[*R:itemname]

--/|Decision 1: Does the row exist?
--?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryFieldsDone

--&Equipped|[*R:equipped]

--/|Decision 2: Should the existing row be included?
--?"[&Equipped]" -eq "1" -or "[&Equipped]" -eq "on"|[
  --&AbilityText|+<br>[&FieldOpen][&ItemName]=[&AtOpen][*R>itemcount][&CloseBrace][&FieldClose]
--]|

--/|Advance whether the row was included or skipped.
--Rnext|
--^BuildInventoryField|

--:InventoryFieldsDone|
```

Notice that `--Rnext` is outside the conditional block.

If it were inside the block, a row that failed the equipped test would never advance to the next row. The script would repeatedly process the same row forever.

That is an example of understanding program flow rather than merely knowing command syntax.

---

# 14. Build the Complete Script in Deliberate Sections

The final script is organised into five sections:

```text
1. Configuration
2. Validation
3. Reserved-syntax construction
4. Macro construction
5. Ability creation
```

Read the section comments and identify which earlier lesson each part came from.

```scard
!script {{
  --#title|Dynamic Ability Builder

  --/|======================================================================
  --/|1. CONFIGURATION
  --/|Change these values when adapting the builder.
  --/|======================================================================

  --&CharacterName|TestSheet
  --&AbilityName|NewAbility

  --/|This Roll20 call intentionally resolves before ScriptCards runs.
  --&CharacterID|@{TestSheet|character_id}


  --/|======================================================================
  --/|2. VALIDATION
  --/|Do not create a duplicate Ability with the same name.
  --/|======================================================================

  --~ExistingAbilityID|system;findability;[&CharacterName];[&AbilityName]
  --?"[&ExistingAbilityID]" -ne "AbilityNotFound"|AbilityAlreadyExists


  --/|======================================================================
  --/|3. CONSTRUCT RESERVED ROLL20 SYNTAX
  --/|These pieces do not become complete Roll20 syntax until runtime.
  --/|======================================================================

  --/|Creates @{
  --&AtOpen|@
  --&AtOpen|+{

  --/|Creates &{template:
  --&TemplateOpen|&
  --&TemplateOpen|+{template:

  --/|Creates {{
  --&FieldOpen|{
  --&FieldOpen|+{

  --/|Creates }}
  --&FieldClose|}
  --&FieldClose|+}

  --/|Creates }
  --&CloseBrace|}


  --/|======================================================================
  --/|4. BUILD THE ABILITY ACTION
  --/|Start with the template header, then append qualifying repeating rows.
  --/|======================================================================

  --&AbilityText|[&TemplateOpen]default[&CloseBrace]
  --&AbilityText|+ [&FieldOpen]name=Equipped Inventory[&FieldClose]

  --Rfirst|[&CharacterID];repeating_inventory

  --:BuildInventoryField|

  --&ItemName|[*R:itemname]

  --/|Stop after Rnext has exhausted the section.
  --?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryFieldsDone

  --&Equipped|[*R:equipped]

  --/|Append only equipped rows.
  --?"[&Equipped]" -eq "1" -or "[&Equipped]" -eq "on"|[
    --&AbilityText|+<br>[&FieldOpen][&ItemName]=[&AtOpen][*R>itemcount][&CloseBrace][&FieldClose]
  --]|

  --/|Always advance after processing an existing row.
  --Rnext|
  --^BuildInventoryField|

  --:InventoryFieldsDone|

  --/|Replace temporary line markers with real line feeds.
  --~AbilityText|string;brtolinefeeds;[&AbilityText]

  --/|Leave this output enabled while developing.
  --+Generated Action|[&AbilityText]


  --/|======================================================================
  --/|5. CREATE THE CHARACTER ABILITY
  --/|======================================================================

  --!ob:NewAbilityID:[&CharacterID]:[&AbilityName]:n|[&AbilityText]

  --+Created|[&AbilityName] was created on [&CharacterName].
  --+Ability ID|[&NewAbilityID]
  --X|


  --/|======================================================================
  --/|VALIDATION FAILURE
  --/|======================================================================

  --:AbilityAlreadyExists|
  --+Stopped|[&AbilityName] already exists on [&CharacterName].
  --+Next Step|Delete or rename the existing test Ability before rebuilding it.
}}
```

---

# 15. Read the Final Script as a Process

Do not memorise the lines. Read the program as a process.

## Configuration

```scard
--&CharacterName|TestSheet
--&AbilityName|NewAbility
--&CharacterID|@{TestSheet|character_id}
```

The script establishes its target.

## Validation

```scard
--~ExistingAbilityID|system;findability;[&CharacterName];[&AbilityName]
--?"[&ExistingAbilityID]" -ne "AbilityNotFound"|AbilityAlreadyExists
```

The built-in function searches for the named Ability.

The conditional branches away from creation when one already exists.

## Syntax Construction

```scard
--&AtOpen|@
--&AtOpen|+{
```

The script creates dangerous Roll20 syntax only after execution has begun.

The other syntax variables use the same technique.

## Initial Macro Text

```scard
--&AbilityText|[&TemplateOpen]default[&CloseBrace]
--&AbilityText|+ [&FieldOpen]name=Equipped Inventory[&FieldClose]
```

The accumulator begins with content that exists exactly once.

## Repeating Loop

```scard
--Rfirst|...
--:BuildInventoryField|
...
--Rnext|
--^BuildInventoryField|
```

The current repeating row changes on each pass.

## End Condition

```scard
--?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryFieldsDone
```

The loop stops only after the repeating section has been exhausted.

## Inclusion Condition

```scard
--?"[&Equipped]" -eq "1" -or "[&Equipped]" -eq "on"|[
```

An existing row contributes text only when it qualifies.

## Dynamic Attribute Construction

```scard
[&AtOpen][*R>itemcount][&CloseBrace]
```

The builder writes a future attribute reference instead of copying the current value.

## Creation

```scard
--!ob:NewAbilityID:[&CharacterID]:[&AbilityName]:n|[&AbilityText]
```

Only after the action is complete does the script create the Ability.

That sequence is the reusable architecture:

```text
Configure
Validate
Prepare
Build
Inspect
Write
```

---

# 16. How to Adapt the Builder to Another Repeating Section

The control flow does not care whether the rows are inventory items, attacks, spells, feats, or something from another game system.

The sheet-specific pieces are:

```text
repeating_inventory
itemname
itemcount
equipped
```

Replace them with:

| Role | What to find on the new sheet |
|---|---|
| Section | The complete repeating-section name |
| Row label | A field that identifies the row |
| Output value | The field the generated macro should read |
| Inclusion field | The field that decides whether to add the row |
| Inclusion value | The value that means “include this row” |

A generic version of the loop is:

```scard
--Rfirst|[&CharacterID];REPEATING_SECTION

--:BuildRow|

--&RowLabel|[*R:ROW_LABEL_FIELD]
--?"[&RowLabel]" -eq "NoRepeatingAttributeLoaded"|RowsDone

--&IncludeValue|[*R:INCLUSION_FIELD]

--?"[&IncludeValue]" -eq "EXPECTED_VALUE"|[
  --&AbilityText|+<br>[&FieldOpen][&RowLabel]=[&AtOpen][*R>OUTPUT_FIELD][&CloseBrace][&FieldClose]
--]|

--Rnext|
--^BuildRow|

--:RowsDone|
```

Do not replace placeholders at random. For each one, explain to yourself what role it serves.

---

# 17. A Better Way to Modify the Script

When changing the final builder, work in this order:

## First: Read the Row

Temporarily replace macro generation with visible output:

```scard
--+Label|[*R:ROW_LABEL_FIELD]
--+Output Value|[*R:OUTPUT_FIELD]
--+Condition Value|[*R:INCLUSION_FIELD]
```

Confirm that the field names and values are correct.

## Second: Test the Condition

Use visible output inside the block:

```scard
--?"[*R:INCLUSION_FIELD]" -eq "EXPECTED_VALUE"|[
  --+Included|This row passed.
--]|
```

Confirm that the intended rows pass.

## Third: Build One Generated Field

Display both forms:

```scard
--+Current Value|[*R:OUTPUT_FIELD]
--+Full Name|[*R>OUTPUT_FIELD]
```

Then construct the deferred attribute call and inspect it.

## Fourth: Append to the Accumulator

Only after the row logic is correct should you append to `[&AbilityText]`.

## Fifth: Create the Ability

Keep this line disabled or removed while debugging:

```scard
--!ob:NewAbilityID:[&CharacterID]:[&AbilityName]:n|[&AbilityText]
```

Add it back only when `--+Generated Action|[&AbilityText]` shows the exact macro you intend to create.

This staged method prevents one error from being buried inside several other features.

---

# 18. Debugging Checklist

## The Ability Contains a Number Instead of `@{attribute}`

The complete `@{...}` call existed before ScriptCards ran.

Construct the opening sequence at runtime:

```scard
--&AtOpen|@
--&AtOpen|+{
```

## The Ability Contains `[*R:field]`

The ScriptCards reference was probably protected or assembled incorrectly and never expanded during the builder run.

Use the repeating reference directly in the append statement.

## The Ability Contains the Current Repeating Value

You used:

```text
[*R:field]
```

when you needed:

```text
[*R>field]
```

The colon form reads now. The greater-than form returns the full attribute name.

## The Loop Never Ends

Check all three parts:

```scard
--Rnext|
--^LoopLabel|
--?"[&RowLabel]" -eq "NoRepeatingAttributeLoaded"|DoneLabel
```

Also ensure that `--Rnext` is not trapped inside a conditional block that some rows can fail.

## The Loop Ends Immediately

The section name may be wrong, the character ID may be wrong, or the character may have no rows in that section.

Display:

```scard
--+Character ID|[&CharacterID]
```

Then test only `--Rfirst` and one repeating reference.

## Rows Exist but Fields Are Blank

The repeating-section name may be correct while the field names are wrong.

Repeating field names are sheet-specific.

## Duplicate Abilities Appear

`--!ob` creates an Ability. It does not replace one with the same name.

Use `system;findability` before creation, as shown in the final script.

## The Generated Template Has Broken Braces

Display every syntax component separately:

```scard
--+AtOpen|[&AtOpen]
--+TemplateOpen|[&TemplateOpen]
--+FieldOpen|[&FieldOpen]
--+FieldClose|[&FieldClose]
--+CloseBrace|[&CloseBrace]
```

Then inspect the completed action before creating the Ability.

---

# 19. Practice Exercises

These are intended to make you change the logic, not just the labels.

## Exercise 1: Include Every Item

Remove the equipped test while retaining the repeating loop.

Explain why `--Rnext` and the end condition are still required.

## Exercise 2: Include Only Items with a Positive Count

Read the current count:

```scard
--&ItemCount|[*R:itemcount]
```

Replace the equipped condition with a numeric comparison.

The generated output should still use `[*R>itemcount]` so the Ability reads the count later.

This exercise demonstrates that the builder can use a value **now** to decide whether to write a reference that will be used **later**.

## Exercise 3: Add Two Values from Each Row

Generate a field containing both count and weight.

First inspect:

```scard
[*R>itemcount]
[*R>itemweight]
```

Then construct two deferred attribute calls in the same generated field.

## Exercise 4: Use Another Repeating Section

Choose attacks, spells, or features.

Before modifying the builder:

1. identify the section name;
2. identify a reliable label field;
3. identify the output field;
4. identify the inclusion field;
5. write a reader script that displays them.

## Exercise 5: Adapt It to Another Character Sheet

Use the generic loop from the previous section.

Keep the ScriptCards control flow and replace only the sheet-specific names and the final macro format.

---

# 20. What You Should Understand Now

After completing the tutorial, you should be able to explain:

- why every ScriptCards statement begins with `--`;
- how the tag and content are separated by `|`;
- how `--&Name|Value` creates a string variable;
- how `--&Name|+More` appends to a string;
- why Roll20 may resolve `@{...}` before ScriptCards runs;
- how constructing `@{` during execution preserves a future attribute call;
- how `--Rfirst` and `--Rnext` manage the current repeating row;
- how labels, conditionals, and branches form a loop;
- why `[*R:field]` and `[*R>field]` solve different problems;
- why the loop-ending condition and row-inclusion condition are separate;
- and why the Ability should be created only after its entire action has been assembled and inspected.

The important lesson is not the final inventory macro.

The important lesson is the construction process:

```text
Break the goal into operations.
Learn the data returned by the sheet.
Build and inspect one piece at a time.
Add control flow only after the simple case works.
Write the finished result only after validating it.
```

That process is what makes the same ScriptCards knowledge transferable to D&D 5E 2014, Pathfinder 2E, and other character sheets.

---

## References

- [ScriptCards documentation on the Roll20 Community Wiki](https://wiki.roll20.net/Script:ScriptCards)
- [ScriptCards in the Roll20 API Scripts repository](https://github.com/Roll20/roll20-api-scripts/tree/master/ScriptCards)
