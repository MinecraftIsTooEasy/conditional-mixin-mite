# conditional-mixin

[![jitpack badge](https://jitpack.io/v/Fallen-Breath/conditional-mixin.svg)](https://jitpack.io/#Fallen-Breath/conditional-mixin)
[![maven badge](https://maven.fallenbreath.me/api/badge/latest/releases/me/fallenbreath/conditional-mixin-fabric)](https://maven.fallenbreath.me/#/releases/me/fallenbreath/conditional-mixin-fabric)

A library mod for using annotation to conditionally apply your mixins to your Minecraft mods

It is available at [jitpack](https://jitpack.io/#Fallen-Breath/conditional-mixin) and [my maven](https://maven.fallenbreath.me/#/releases/me/fallenbreath/conditional-mixin-fabric)

## Example Usages

conditional-mixin supports multiple mod loader platforms, including:

| Platform | Requirements                   |
|----------|--------------------------------|
| Fabric   | Fabric loader `>=0.10.4`       |
| Forge    | Forge >= `28` (mc1.14.4+)      |
| NeoForge | NeoForge >= `20.5` (mc1.20.5+) |

conditional-mixin does not require any extra dependencies

### Import

Import conditional-mixin in gradle

Artifact ID table:

| Platform | Artifact ID                  |
|----------|------------------------------|
| Fabric   | `conditional-mixin-fabric`   |
| Forge    | `conditional-mixin-forge`    |
| NeoForge | `conditional-mixin-neoforge` |

The following documents will use Fabric as the example. For other platforms, just replace the artifact ID accordingly

#### From jitpack

```groovy
repositories {
    maven { url 'https://jitpack.io' }
}

dependencies {
    modImplementation 'com.github.Fallen-Breath.conditional-mixin:conditional-mixin-fabric:0.6.4'

    // Suggested, to bundle it into your mod jar. Choose a method your build system provides
    include 'com.github.Fallen-Breath.conditional-mixin:conditional-mixin-fabric:0.6.4'
}
```

#### From my maven

```groovy
repositories {
    maven { url 'https://maven.fallenbreath.me/releases' }
}

dependencies {
    modImplementation 'me.fallenbreath:conditional-mixin-fabric:0.6.4'

    // Suggested, to bundle it into your mod jar. Choose a method your build system provides
    include 'me.fallenbreath:conditional-mixin-fabric:0.6.4'
}
```

### Integrate

You need to create a [mixin config plugin](https://github.com/SpongePowered/Mixin/blob/master/src/main/java/org/spongepowered/asm/mixin/extensibility/IMixinConfigPlugin.java)
to provide the ability to control mixin applications. Then, there are 2 ways to integrate it with conditional-mixin:

#### The simplest way

Let your mixin config plugin class inherit [`RestrictiveMixinConfigPlugin`](common/src/main/java/me/fallenbreath/conditionalmixin/api/mixin/RestrictiveMixinConfigPlugin.java)

The `RestrictiveMixinConfigPlugin` will disable those mixins that don't satisfy with the annotated restriction in its `shouldApplyMixin` method

```java
import me.fallenbreath.conditionalmixin.api.mixin.RestrictiveMixinConfigPlugin;

public class MyMixinConfigPlugin extends RestrictiveMixinConfigPlugin
{
    // ...
}
```

Specify the mixin config plugin class in your mixin meta json, if you have not done that yet:

```yaml
"plugin": "my.mod.MyMixinConfigPlugin",
```

#### The universal way

If you have already written a custom mixin plugin and don't want to make your plugin class inherit from something else,
you can directly use the [`RestrictionChecker`](common/src/main/java/me/fallenbreath/conditionalmixin/api/checker/RestrictionChecker.java) provided by conditional-mixin

```java
import me.fallenbreath.conditionalmixin.api.checker.RestrictionChecker;
import me.fallenbreath.conditionalmixin.api.checker.RestrictionCheckers;

public class MyMixinConfigPlugin
{
	private final RestrictionChecker restrictionChecker = RestrictionCheckers.memorized();
	
	// See RestrictiveMixinConfigPlugin for usages of a RestrictionChecker
}
```

(optional) To erase `@Restriction` annotations on output mixined classes, you can use
`me.fallenbreath.conditionalmixin.api.mixin.AnnotationCleaner` and call `AnnotationCleaner.create(Restriction.class)` to get an annotation cleaner

See also: source code of the [`RestrictiveMixinConfigPlugin`](common/src/main/java/me/fallenbreath/conditionalmixin/api/mixin/RestrictiveMixinConfigPlugin.java) class

### Annotate your mixins

Now, you can annotate your mixins like these:

```java
@Restriction(
        require = {
                @Condition("some_mod"),
                @Condition(value = "another_mod", versionPredicates = "2.0.x"),
                @Condition(value = "random_mod", versionPredicates = {">=1.0.1 <1.2", ">=2.0.0"}),
        }
)
@Mixin(SomeClass.class)
public abstract class SomeClassMixin
{
    // ...
}
```

or

```java
@Restriction(
        require = @Condition(type = Condition.Type.MIXIN, value = "my.mod.mixin.ImportantMixin"),  // should be another mixin in your mod
        conflict = @Condition("bad_mod")
)
@Mixin(AnotherClass.class)
public abstract class AnotherClassMixin
{
    // ...
}
```

You can also define your own tester class for more complicated condition testing

```java
@Restriction(
        require = @Condition(type = Condition.Type.TESTER, tester = MyConditionTester.class)
)
@Mixin(RandomClass.class)
public abstract class RandomClassMixin
{
    // ...
}

public class MyConditionTester implements ConditionTester
{
	@Override
	public boolean isSatisfied(String mixinClassName)
	{
		// More complicated checks go here
		return mixinClassName.length() % 2 == 0;
	}
}
```

## Notes

If you are upgrading conditional-mixin from older version, and your mixin plugin class inherits from `RestrictiveMixinConfigPlugin`, 
make sure to check if you have overwritten some methods in your mixin plugin class,
since class `RestrictiveMixinConfigPlugin` might implement more methods of `IMixinConfigPlugin` in newer versions.
e.g. `RestrictiveMixinConfigPlugin#preApply` and `RestrictiveMixinConfigPlugin#postApply` are added in `v0.2.0`
for being able to automatically remove the `@Restriction` in the merged target class
