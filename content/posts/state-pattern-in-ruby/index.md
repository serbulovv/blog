+++
date = '2026-09-20T00:00:00+02:00'
title = 'State Pattern in Ruby'
author = 'Genry Wood'
tags = ['pattern', 'state', 'ruby', 'rpg', 'game', 'console']
+++

![Post Logo](./header.png)

# State Pattern in Ruby: Replacing a case/when Menu in My Console RPG

## My menu got out of hand

I have a side project, a console RPG written in pure Ruby. Nothing fancy: you pick an option in the menu, go to a dungeon, fight a monster and collect the loot.
In the beginning the whole menu lived in one Menu class with a case inside. It's horrible looked like this.

```ruby
module Modules
  class Menu
    MAIN_MENU_FIELDS = [{ name: 'Start', position: 1 }, { name: 'Exit', position: 2 }]
    DUNGEONS_MENU_FIELDS = [
      { name: 'Back', position: 1, id: 1 },
      { name: 'Forest', position: 2, id: 2 }
    ]

    def initialize
      @player = Player.new
      @current_menu = :main
    end

    def run
      loop do
        case @current_menu
        when :main
          render_main_menu
        when :dungeons
          render_dungeons_menu
        when :exit
          break
        end
      end
    end

    private

    def render_main_menu
      clear_menus

      MAIN_MENU_FIELDS.each do |field|
        puts "#{field[:position]}. #{field[:name]}"
      end

      print "#{Helpers::Locales::CHOOSE_MENU_OPTION}: "
      input_result = gets.chomp.to_i

      case input_result
      when 1
        @current_menu = :dungeons
      when 2
        @current_menu = :exit
      else
        puts Helpers::Locales::WRONG_MENU_OPTION_ERROR
      end
    end

    # some code
  end
end
```

While the menu was small, it worked fine. I would even say that for me it was the most obvious way to do it, so I didn't overthink it.

The trouble started when I began adding new screens: a fight, an inventory, crafting, equipment. Every new screen meant going back into the same case and adding one more when. That's editing code that already works, and the more branches it had, the harder it was to see which of them belonged to which screen.

So I started looking for a suitable pattern for this and that's when I decided to look at the State pattern.

## The Solution: State pattern and its roles (Context, Base State, Concrete States)

I took the State pattern example from [refactoring.guru](https://refactoring.guru/design-patterns/state/ruby/example) and mapped it onto my menu almost 1 to 1. The idea is simple: every screen becomes its own class, and the object that runs the game only holds the current screen and delegates everything to it.

So the setup has three roles:

**Context**:
- The only class the rest of the game talks to. Holds the current state and delegates work to it.
- My classes: `Modules::MenuContext`.

**Base State**:
- A contract that every screen has to follow.
- My classes: `Modules::States::MenuState`.

**Concrete States**:
- One screen, one class. Rendering and input handling for that screen live together.
- My classes: `Modules::States::MainMenuState`,` Modules::States::DungeonsMenuState`.

### Context

```ruby
# frozen_string_literal: true

module Modules
  class MenuContext
    attr_reader :state

    def initialize(state)
      transition_to(state)
    end

    def transition_to(state)
      @state = state
      @state.context = self if @state
    end

    def run
      while @state
        @state.render
        input = gets.chomp.to_i
        @state.handle_input(input)
      end
    end
  end
end
```
This is the class that replaced my old Menu. It doesn't know how many screens exist or what they do. It keeps a reference to the current state and passes render and handle_input to it

Key points of the class:
- it keeps the current state in `@state`;
- it delegates rendering and input handling to that state;
- the loop lives while `@state` is not nil, so leaving the game is just transition_to(nil).

### Base State

```ruby
# frozen_string_literal: true

module Modules
  module States
    class MenuState
      attr_accessor :context

      # @abstract
      def render
        raise NotImplementedError, "#{self.class} has not implemented method '#{__method__}'"
      end

      # @abstract
      def handle_input(input)
        raise NotImplementedError, "#{self.class} has not implemented method '#{__method__}'"
      end

      private

      def clear_menus
        puts `clear`
      end
    end
  end
end
```

The base class is a contract (I usualy call that abstract class): any state must have `render` and `handle_input`, so the context can call them without caring which screen it holds. `NotImplementedError` is there for a simple reason. If I create a new state and forget one of the methods, I get a clear error right away instead of a silent nil somewhere at runtime.

`attr_accessor :context` is the link back to the context.

### Concrete States

```ruby
# frozen_string_literal: true

require_relative 'menu_state'

module Modules
  module States
    class MainMenuState < MenuState
      FIELDS = [{ name: 'Start', position: 1 }, { name: 'Exit', position: 2 }].freeze

      def render
        clear_menus

        FIELDS.each do |field|
          puts "#{field[:position]}. #{field[:name]}"
        end

        print "#{Helpers::Locales::CHOOSE_MENU_OPTION}: "
      end

      def handle_input(input)
        case input
        when 1
          context.transition_to(DungeonsMenuState.new)
        when 2
          context.transition_to(nil)
        else
          puts Helpers::Locales::WRONG_MENU_OPTION_ERROR
        end
      end
    end
  end
end
```

Everything about the main menu is now in one file: what is drawn and what every option does. `DungeonsMenuState` looks the same, just with its own fields and its own options.
The case input is still here, by the way. What disappeared is the other case, the one that decided which screen we are on. Now every screen decides that for itself.
Adding a new screen means adding a new class. `MenuContext` stays untouched, and only the state that leads to the new screen gets one more when.


## Transitions between states

The part I liked the most is who decides when to switch the screen. In my old Menu the class itself did it through `@current_menu`. Here it's the other way around: a state decides where to go next, and the context only executes it.

```ruby
# inside any state
context.transition_to(DungeonsMenuState.new)
```

And here is what the context does with it.

```ruby
def transition_to(state)
  @state = state
  @state.context = self if @state
end
```

Two things happen here:
- the new state becomes the current one;
- it gets a back-reference to the context, so it can trigger the next transition on its own.

If `@state` part covers the exit. transition_to(nil) sets `@state` to nil, and the while `@state` loop in run simply ends.
One detail worth noticing: a state knows the other state classes (it has to, to create them), but it never holds a live instance of another state. Every transition creates a new object.
There is also no big map of transitions anywhere. If I want to know where the player can go from the main menu, I open `main_menu_state.rb` and see it right there.

## Keeping states thin: FightState and BattleProcess

The last thing I want to show is the fight screen. It's a state like all the others, so the first idea was to put the whole battle inside it. I decided to keep it as thin as the rest of the states, and moved everything about the fight into a separate `BattleProcess` class.

```ruby
# frozen_string_literal: true

require_relative 'menu_state'
require_relative '../battle/battle_process'

module Modules
  module States
    class FightState < MenuState
      def initialize(player, enemy)
        @player = player
        @enemy = enemy
      end

      def render
        battle_process = Battle::BattleProcess.new(@player, @enemy)
        battle_process.run
      end

      def handle_input(_input)
        context.transition_to(MainMenuState.new)
      end
    end
  end
end
```

`FightState` knows nothing about the battle itself. It starts the process, waits for any key after the fight and moves the player to the next screen. Everything else lives in `BattleProcess`

It's the same idea as a thin controller in Rails. The state coordinates, the service does the work. As a bonus, I can change how the battle works without opening a single state.

## Conclusions

With this article I wanted to show how I moved a growing `case/when` menu to the State pattern in a small console RPG, following the `refactoring.guru` example.

What I got out of it:
- `MenuContext` stays small and knows nothing about the screens;
- every screen is one class, so I don't have to search for its logic in a big method;
- transitions live inside the states, so there is no central map of them to maintain;
- adding a new screen means adding a new class, and only the state that leads to it gets one more when;
- heavy logic, like the fight, lives outside the state in its own class.

You can find the whole project in the [repository](https://github.com/serbulovv/console-rig).
