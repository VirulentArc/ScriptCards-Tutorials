# Building Character Abilities with ScriptCards

This tutorial builds one concept at a time:

1. Create a Character Ability.
2. Write a roll template containing unresolved attributes.
3. Generate template fields from every row of a repeating section.
4. Include only rows that meet a condition.
5. Combine everything into one reusable builder.

The examples use:

```text
Character: TestSheet
Ability: NewAbility
```

The repeating-section examples use the **D&D 5E 2014 inventory section**. The ScriptCards logic is not tied to D&D, but repeating-section and field names are defined by each character sheet.

---

## The Two Stages of Expansion

There are two different systems involved.

### Roll20 expands these before ScriptCards runs

```text
@{TestSheet|character_id}
@{selected|strength}
?{Choose something}
```

### ScriptCards expands these while the card runs

```text
[&StringVariable]
[*S:strength]
[*R:itemname]
[*R>itemname]
```

This matters because writing this directly inside the ScriptCard:

```text
@{strength}
```

usually causes Roll20 to resolve it while running the builder. The generated Ability would receive the current Strength value instead of the text `@{strength}`.

To preserve it, construct `@{` after ScriptCards begins running:

```scard
--&AtOpen|@
--&AtOpen|+{
```

The resulting value of `[&AtOpen]` is:

```text
@{
```

Roll20 never saw those two characters together in the original command, so it could not resolve the attribute early.

---

# Lesson 1: Create a Basic Ability

Create a Roll20 macro containing:

```scard
!script {{
  --#title|Ability Writer

  --/|This lookup is supposed to resolve now.
  --&CharacterID|@{TestSheet|character_id}

  --/|Create the Ability.
  --!ob:NewAbilityID:[&CharacterID]:NewAbility:n|This is a test

  --+Created|NewAbility was created on TestSheet.
  --+Ability ID|[&NewAbilityID]
}}
```

After running it, open `TestSheet` and look under **Attributes & Abilities**.

You should find:

```text
Ability name: NewAbility
Action: This is a test
```

The creation command has this structure:

```scard
--!ob:ReturnVariable:CharacterID:AbilityName:IsTokenAction|Action
```

In this example:

```text
ReturnVariable = NewAbilityID
CharacterID    = [&CharacterID]
AbilityName    = NewAbility
IsTokenAction  = n
Action         = This is a test
```

Use `y` instead of `n` when the Ability should be a token action.

> **Important:** Running this first example repeatedly creates duplicate Abilities. It is deliberately minimal. Later examples find and update the existing Ability instead.

---

# Lesson 2: Write Unresolved Attributes into the Ability

Suppose the generated Ability should contain:

```text
&{template:default} {{name=Attribute Test}} {{Character=@{character_name}}} {{Strength=@{strength}}}
```

The `@{character_name}` and `@{strength}` references must remain unresolved until someone runs `NewAbility`.

Avoid putting literal roll-template braces into the outer ScriptCard. Build the special pieces separately:

```scard
!script {{
  --#title|Ability Writer

  --&CharacterID|@{TestSheet|character_id}

  --/|Find the Ability created in Lesson 1.
  --~AbilityID|system;findability;TestSheet;NewAbility
  --?"[&AbilityID]" -eq "AbilityNotFound"|AbilityMissing

  --/|Build the reserved Roll20 syntax after execution begins.
  --&AtOpen|@
  --&AtOpen|+{

  --&TemplateOpen|&
  --&TemplateOpen|+{template:

  --&DoubleOpen|{
  --&DoubleOpen|+{

  --&DoubleClose|}
  --&DoubleClose|+}

  --&RightBrace|}

  --/|Assemble the complete Ability action.
  --&MacroText|[&TemplateOpen]default[&RightBrace] [&DoubleOpen]name=Attribute Test[&DoubleClose] [&DoubleOpen]Character=[&AtOpen]character_name[&RightBrace][&DoubleClose] [&DoubleOpen]Strength=[&AtOpen]strength[&RightBrace][&DoubleClose]

  --/|Replace the existing Ability action.
  --!ability:[&AbilityID]|action:[&MacroText]

  --+Updated|NewAbility now contains unresolved attribute calls.
  --X|

  --:AbilityMissing|
  --+Error|Run Lesson 1 first so NewAbility exists.
}}
```

Open `NewAbility` after running the builder. Its action should contain:

```text
&{template:default} {{name=Attribute Test}} {{Character=@{character_name}}} {{Strength=@{strength}}}
```

It should **not** contain the character's current name or Strength score.

When `NewAbility` itself is run later, the Ability belongs to `TestSheet`, so unqualified references such as these use that character:

```text
@{character_name}
@{strength}
```

## Why Not Use a Literal `${ ... $}` Block?

ScriptCards literal blocks protect text from ScriptCards line splitting and variable replacement. They are useful when embedding text containing ScriptCards commands such as `--+`.

They do not solve the earlier Roll20 attribute-expansion stage by themselves. They would also prevent the ScriptCards variables inside the block from being assembled normally.

For this task, constructing the reserved characters from pieces is the more useful technique.

---

# Lesson 3: Repeat Fields Until There Are No More Rows

The D&D 5E 2014 inventory section is:

```text
repeating_inventory
```

This example uses:

```text
itemname
itemcount
```

The generated Ability will contain one template field for every inventory row.

```scard
!script {{
  --#title|Repeating Ability Writer

  --&CharacterID|@{TestSheet|character_id}
  --~AbilityID|system;findability;TestSheet;NewAbility
  --?"[&AbilityID]" -eq "AbilityNotFound"|AbilityMissing

  --/|Construct reserved Roll20 syntax.
  --&AtOpen|@
  --&AtOpen|+{

  --&TemplateOpen|&
  --&TemplateOpen|+{template:

  --&DoubleOpen|{
  --&DoubleOpen|+{

  --&DoubleClose|}
  --&DoubleClose|+}

  --&RightBrace|}

  --/|Start the generated macro.
  --&MacroText|[&TemplateOpen]default[&RightBrace] [&DoubleOpen]name=Inventory[&DoubleClose]

  --/|Load the first inventory row.
  --Rfirst|[&CharacterID];repeating_inventory

  --:InventoryLoop|
  --&ItemName|[*R:itemname]

  --/|Rfirst or Rnext returns this when no row is loaded.
  --?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryDone

  --/|Append one generated template field.
  --&MacroText|+<br>[&DoubleOpen][&ItemName]=[&AtOpen][*R>itemcount][&RightBrace][&DoubleClose]

  --/|Load the next row and repeat.
  --Rnext|
  --^InventoryLoop|

  --:InventoryDone|

  --/|Turn visible <br> separators into real line breaks.
  --~MacroText|string;brtolinefeeds;[&MacroText]

  --!ability:[&AbilityID]|action:[&MacroText]

  --+Updated|NewAbility now contains one field per inventory row.
  --X|

  --:AbilityMissing|
  --+Error|Run Lesson 1 first so NewAbility exists.
}}
```

## The Important Repeating-Row Distinction

This reference returns the value right now:

```scard
[*R:itemcount]
```

For example, it might return:

```text
3
```

This reference returns the complete attribute name:

```scard
[*R>itemcount]
```

For example:

```text
repeating_inventory_-AbCdEf123_itemcount
```

The tutorial combines that full attribute name with `[&AtOpen]`:

```scard
[&AtOpen][*R>itemcount][&RightBrace]
```

That produces:

```text
@{repeating_inventory_-AbCdEf123_itemcount}
```

Therefore, the generated Ability reads the item count when the Ability is used rather than permanently copying the count that existed when the builder ran.

The item name is handled differently:

```scard
[&ItemName]
```

That copies the current name into the generated template as its label. Renaming an item therefore requires rerunning the builder. Its count remains dynamic.

---

# Lesson 4: Include a Line Only When a Value Qualifies

Suppose only equipped inventory items should appear.

On the D&D 5E 2014 sheet, the repeating inventory field is:

```text
equipped
```

Place a conditional around the line that appends each item:

```scard
--&Equipped|[*R:equipped]

--?"[&Equipped]" -eq "1" -or "[&Equipped]" -eq "on"|[
  --&MacroText|+<br>[&DoubleOpen][&ItemName]=[&AtOpen][*R>itemcount][&RightBrace][&DoubleClose]
--]|
```

The row is still visited either way. The conditional decides whether that row contributes text to the generated Ability.

This is different from the loop-ending condition:

```scard
--?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryDone
```

The two conditions serve different purposes:

```text
No row exists:
Stop looping.

Row exists but equipped is false:
Continue looping, but do not add that row.
```

Other sheets may store checkboxes as `1`, `on`, `true`, or another value. Inspect the actual attribute before choosing the comparison.

---

# Final Combined Example

This version:

- targets `TestSheet`;
- creates `NewAbility` when it does not exist;
- updates it when it already exists;
- scans every D&D 5E 2014 inventory row;
- includes only equipped items;
- copies item names into the template;
- leaves item-count attributes unresolved.

```scard
!script {{
  --#title|Dynamic Ability Builder

  --/|Configuration
  --&CharacterName|TestSheet
  --&AbilityName|NewAbility

  --/|This named lookup intentionally resolves before ScriptCards runs.
  --&CharacterID|@{TestSheet|character_id}

  --/|Construct reserved Roll20 syntax.
  --&AtOpen|@
  --&AtOpen|+{

  --&TemplateOpen|&
  --&TemplateOpen|+{template:

  --&DoubleOpen|{
  --&DoubleOpen|+{

  --&DoubleClose|}
  --&DoubleClose|+}

  --&RightBrace|}

  --/|Begin the generated Ability action.
  --&MacroText|[&TemplateOpen]default[&RightBrace] [&DoubleOpen]name=Equipped Inventory[&DoubleClose]
  --&IncludedCount|0

  --/|Load and process the repeating inventory section.
  --Rfirst|[&CharacterID];repeating_inventory

  --:InventoryLoop|
  --&ItemName|[*R:itemname]

  --/|No loaded row means the loop is finished.
  --?"[&ItemName]" -eq "NoRepeatingAttributeLoaded"|InventoryDone

  --&Equipped|[*R:equipped]

  --/|Only append equipped rows.
  --?"[&Equipped]" -eq "1" -or "[&Equipped]" -eq "on"|[
    --&MacroText|+<br>[&DoubleOpen][&ItemName]=[&AtOpen][*R>itemcount][&RightBrace][&DoubleClose]
    --&IncludedCount|[= [&IncludedCount] + 1]
  --]|

  --Rnext|
  --^InventoryLoop|

  --:InventoryDone|

  --/|Convert the construction markers to real line feeds.
  --~MacroText|string;brtolinefeeds;[&MacroText]

  --/|Find or create the destination Ability.
  --~AbilityID|system;findability;[&CharacterName];[&AbilityName]

  --?"[&AbilityID]" -eq "AbilityNotFound"|[
    --!ob:AbilityID:[&CharacterID]:[&AbilityName]:n|Temporary contents
  --]|

  --/|Write the completed action.
  --!ability:[&AbilityID]|action:[&MacroText]

  --+Complete|[&AbilityName] was written to [&CharacterName].
  --+Included Rows|[&IncludedCount]
}}
```

The resulting Ability will resemble:

```text
&{template:default}
{{name=Equipped Inventory}}
{{Longsword=@{repeating_inventory_-RowID_itemcount}}}
{{Potion of Healing=@{repeating_inventory_-AnotherRowID_itemcount}}}
```

The real generated row IDs will be used.

---

# Adapting This to Another Sheet

The ScriptCards structure remains the same. Replace only the sheet-specific names.

```text
repeating_inventory   → the sheet's repeating-section name
itemname              → the field that identifies each row
itemcount             → the field the generated macro should read
equipped               → the field controlling inclusion
```

For example, a Pathfinder sheet might use completely different section and field names, but the loop remains:

```scard
--Rfirst|[&CharacterID];REPEATING_SECTION

--:RowLoop|
--&RowName|[*R:IDENTIFYING_FIELD]
--?"[&RowName]" -eq "NoRepeatingAttributeLoaded"|RowsDone

--?"[*R:CONDITION_FIELD]" -eq "EXPECTED_VALUE"|[
  --&MacroText|+GENERATED TEXT
--]|

--Rnext|
--^RowLoop|

--:RowsDone|
```

Use a field expected to exist on every valid row as the loop-ending test. A name, label, or type field is usually better than an optional description or modifier.

---

# Practical Debugging

During development, display the generated string before writing it:

```scard
--+Generated Action|[&MacroText]
```

You can also temporarily skip this line:

```scard
--!ability:[&AbilityID]|action:[&MacroText]
```

That lets you inspect the output without modifying the Ability.

When checking a repeating section, display the current value and full name together:

```scard
--+Current Value|[*R:itemcount]
--+Attribute Name|[*R>itemcount]
```

This quickly shows whether you are copying a value or constructing a deferred reference.

---

# The Reusable Pattern

The complete process is:

```text
1. Resolve the destination character.
2. Construct reserved Roll20 syntax from pieces.
3. Start the macro text in a string variable.
4. Load the first repeating row.
5. Stop when NoRepeatingAttributeLoaded is returned.
6. Test whether the current row should be included.
7. Append generated text when it qualifies.
8. Load the next row.
9. Find or create the destination Ability.
10. Write the completed string to its action property.
```

Once that pattern is understood, the same builder can produce inventory summaries, attack menus, spell lists, resource displays, proficiency lists, or system-specific roll-template macros.

---

## Further Reading

- [ScriptCards documentation on the Roll20 Community Wiki](https://wiki.roll20.net/Script:ScriptCards)
- [ScriptCards source and releases in the Roll20 API Scripts repository](https://github.com/Roll20/roll20-api-scripts/tree/master/ScriptCards)
