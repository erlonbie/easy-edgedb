---
tags: Backlinks, Schema Cleanup
---

# Chapter 19 - Dracula escapes

> Van Helsing hypnotizes Mina, who is now half vampire and can feel Dracula. He asks her questions:
>
> “Where are you now?”
>
> “I do not know. It is all strange to me!”
>
> “What do you see?”
>
> “I can see nothing; it is all dark.”
>
> “What do you hear?”
>
> “The water... and little waves.”
>
> “Then you are on a ship?”
>
> “Oh, yes!”
>
> Now our heroes know that Dracula has escaped on a ship with his last box and is going back to Transylvania. Van Helsing and Mina go to Castle Dracula, while the others go to Varna to try to catch the ship when it arrives. Jonathan Harker just sits there and sharpens his knife. He looks like a different person now. Jonathan has lost all of his fear and only wants to kill Dracula and save his wife. But where is the ship? Every day they wait... and then one day, they get a message: the ship arrived at Galatz up the river, not Varna! Are they too late? They rush off up the river to try to find Dracula.

## Adding some new types

We have another ship moving around the map in this chapter. Last time, we made a `Ship` type that looks like this:

```sdl
type Ship extending HasNameAndCoffins {
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

That's not bad, but we can build on it a bit more in this chapter. Right now, it doesn't have any information about visits made by which ship to where. Let's make a quick type that contains all the information on ship visits. Each visit will have a link to a `Ship` and a `Place`, and a `cal::local_date`. It looks like this:

```sdl
type ShipVisit {
  required ship: Ship;
  required place: Place;
  required date: cal::local_date;
}
```

This new ship that Dracula is on is called the `Czarina Catherine` (a Czarina is a Russian queen). Let's do a migration, and then insert a few visits from the ships we know. The other ship that we saw in the book was called The Demeter and left from Varna towards Whitby.

But first we'll insert a new `Ship` and two new places (`City` objects) so we can link them. We know the name of the ship and that there is one coffin in it: Dracula's last coffin. But we don't know about the crew, so we'll just insert this information:

```edgeql
insert Ship {
  name := 'Czarina Catherine',
  coffins := 1,
};
```

After that we have the two cities of Varna and Galatz. We'll put them in at the same time:

```edgeql
for city in {'Varna', 'Galatz'}
union (
  insert City {
    name := city
  }
);
```

The captain's book from the Demeter has a lot of other places too, so let's look at a few of them. The Demeter also passed through the Bosphorus. That is the strait in Turkey that connects the Black Sea to the Aegean Sea, and divides Europe from Asia, so it's not a city. We can use the `OtherPlace` type for it, which is also the type we added some annotations to in Chapter 14. Remember how to look at the message inside an annotation? It looks like this:

```edgeql
select (introspect OtherPlace) {
  name,
  annotations: {
    @value
  }
};
```

Let's look at the output to see what we wrote before to make sure that we should use it:

```
{
  schema::ObjectType {
    name: 'default::OtherPlace',
    annotations: {
      schema::Annotation {
        @value: 'A place with under 50 buildings - hamlets, small villages, etc.',
      },
      schema::Annotation {
        @value: 'Castles and castle towns do not count! Use the Castle type for that',
      },
    },
  },
}
```

Well, the Bosphorus isn't a castle and it isn't actually a place with buildings, so it should work. Soon we will create a `Region` type so that we can have a `Country` → `Region` → `City` layout. In that case, `OtherPlace` might be linked to from `Country` or `Region`. But in the meantime we'll add it without being linked to anything:

```edgeql
insert OtherPlace {
  name := 'Bosphorus'
};
```

That was easy. Now we can put the ship visits in.

```edgeql
for visit in {
    ('The Demeter', 'Varna', '1893-07-06'),
    ('The Demeter', 'Bosphorus', '1893-07-11'),
    ('The Demeter', 'Whitby', '1893-08-08'),
    ('Czarina Catherine', 'London', '1893-10-05'),
    ('Czarina Catherine', 'Galatz', '1893-10-28')
  }
union (
  insert ShipVisit {
    ship  := (select Ship filter .name = visit.0),
    place := (select Place filter .name = visit.1),
    date  := <cal::local_date>visit.2
  }
);
```

With this data, now our game can have certain ships in cities at certain dates. For example, imagine that a character has entered the city of Galatz. If the date is 28 October 1893, we can see if there are any ships in town:

```edgeql
select ShipVisit {
  ship: {
    name
  },
  place: {
    name
  },
  date
} filter .place.name = 'Galatz' and .date = <cal::local_date>'1893-10-28';
```

And it looks like there is a ship in town! It's the Czarina Catherine.

```
{
  default::ShipVisit {
    ship: default::Ship {name: 'Czarina Catherine'},
    place: default::City {name: 'Galatz'},
    date: <cal::local_date>'1893-10-28',
  },
}
```

While we're doing this, let's practice computed backlinks again on our visits. Our `Ship` type doesn't contain any information on the places that it visited:

```sdl
type Ship extending HasNameAndCoffins {
  multi sailors: Sailor;
  multi crew: Crewman;
}
```

But if we use a backlink we can see all of this information. Let's show all of the `Ship` objects, their name, sailors and crew, plus the place names and dates of all the visits they made.

```edgeql
select Ship {
 name,
 sailors: { name },
 crew: { name },

 # All of the ShipVisit objects with a ship property
 # that link to a Ship object
 visits := .<ship[is ShipVisit] {
   place: {name},
   date
  }
};
```

The output is pretty nice! It's basically a report on every ship, who is on them and where and when they visited.

```
{
  default::Ship {
    name: 'The Demeter',
    sailors: {
      default::Sailor {name: 'The Captain'},
      default::Sailor {name: 'Petrofsky'},
      default::Sailor {name: 'The Second Mate'},
      default::Sailor {name: 'The Cook'},
    },
    crew: {
      default::Crewman {name: 'Crewman 1'},
      default::Crewman {name: 'Crewman 2'},
      default::Crewman {name: 'Crewman 3'},
      default::Crewman {name: 'Crewman 4'},
      default::Crewman {name: 'Crewman 5'},
    },
    visits: {
      default::ShipVisit {
        place: default::City {name: 'Varna'},
        date: <cal::local_date>'1893-07-06',
      },
      default::ShipVisit {
        place: default::OtherPlace {name: 'Bosphorus'},
        date: <cal::local_date>'1893-07-11',
      },
      default::ShipVisit {
        place: default::City {name: 'Whitby'},
        date: <cal::local_date>'1893-08-08',
      },
    },
  },
  default::Ship {
    name: 'Czarina Catherine',
    sailors: {},
    crew: {},
    visits: {
      default::ShipVisit {
        place: default::City {name: 'London'},
        date: <cal::local_date>'1893-10-05',
      },
      default::ShipVisit {
        place: default::City {name: 'Galatz'},
        date: <cal::local_date>'1893-10-28',
      },
    },
  },
}
```

## More cleaning up the schema

The heroes of the story found out about when the Czarina Catherine arrived thanks to a telegram by a company in the city of Varna that told them. Here's what it said:

```
28 October.—Telegram. Rufus Smith, London, to Lord Godalming, care H. B. M. Vice Consul, Varna.

“Czarina Catherine reported entering Galatz at one o’clock to-day.”
```

Speaking of time, remember our `Time` type? We made it so that we could enter a string and get some helpful information in return. It looks like this:

```sdl
type Time { 
  required clock: str; 
  property clock_time := <cal::local_time>.clock; 
  property hour := .clock[0:2]; 
  property vampires_are := 
    SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
    else SleepState.Awake;
} 
```

Now that we know that the time was one o'clock, let's put that into a query to display some extra information on the visit. See if anything feels weird to you about this query.

```edgeql
with time := (
  insert Time {
    clock := '13:00:00'
  }
)
select ShipVisit {
  place: {
    name
  },
  ship: {
    name
  },
  date,
  time_ := time {
    clock,
    clock_time,
    hour,
    vampires_are
  },
} filter .place.name = 'Galatz';
```

The output looks pretty good, including whether vampires were awake or asleep when the ship arrived:

```
{
  default::ShipVisit {
    place: default::City {name: 'Galatz'},
    ship: default::Ship {name: 'Czarina Catherine'},
    date: <cal::local_date>'1893-10-28',
    time_: default::Time {
      clock: '13:00:00',
      clock_time: <cal::local_time>'13:00:00',
      hour: '13',
      vampires_are: Asleep,
    },
  },
}
```

However, the problem is that we now have a random `Time` type floating around that is not linked to anything. The `Time` type as it stands is essentially just an expression, and would be better as an alias or a function. As a function it would look like this and would not require inserting an object:

```sdl
function time(clock: str) -> tuple<cal::local_time, str, SleepState> using (
  with local := <cal::local_time>clock,
  hour := clock[0:2],
  vampires_are := SleepState.Asleep if <int16>hour > 7 and <int16>hour < 19
    else SleepState.Awake,
    select(local, hour, vampires_are)
);
```

So that could be an option for later. But for now, let's just steal all the properties from `Time` to improve the `ShipVisit` type instead:

```sdl
type ShipVisit {
  required ship: Ship;
  required place: Place;
  required date: cal::local_date;
  clock: str;
  property clock_time := <cal::local_time>.clock;
  property hour := .clock[0:2];
  property vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
        else SleepState.Awake;
}
```

Then do a schema migration and update the visit to Galatz to give it a `clock`:

```edgeql
update ShipVisit filter .place.name = 'Galatz'
set {
  clock := '13:00:00'
};
```

Then we'll query this `ShipVisit` to Galatz again. Let's add a computed property for fun, assuming that it took two hours, five minutes and ten seconds for Arthur to get the telegram. We'll cast the string to a `cal::local_time` and then add a `duration` to it.

```edgeql
with duration := <duration>'2 hours, 5 minutes, 10 seconds',
select ShipVisit {
  place: {name},
  ship: {name},
  date,
  clock,
  when_arthur_got_the_telegram := <cal::local_time>.clock + duration,
  hour,
  vampires_are
} filter .place.name = 'Galatz';
  
```

And now we get all the output that the `Time` type gave us before, plus our extra info about when Arthur got the telegram:

```
{
  default::ShipVisit {
    place: default::City {name: 'Galatz'},
    ship: default::Ship {name: 'Czarina Catherine'},
    date: <cal::local_date>'1893-10-28',
    clock: '13:00:00',
    hour: '13',
    vampires_are: Asleep,
    when_arthur_got_the_telegram: <cal::local_time>'15:05:10',
  },
}
```

Since we are looking at `Place` again, now we can finish up the chapter by filling out the map with the `Region` type that we discussed. We'll have the `Region` type link to cities, castles and other places, and then change our `Country` type to link to `Region` objects.

```sdl
type Country extending Place {
  multi regions: Region;
}

type Region extending Place {
  multi cities: City;
  multi other_places: OtherPlace;
  multi castles: Castle;
}
```

That connects our types based on `Place` quite well.

Now let's do a medium-sized entry that has `Country`, `Region`, and `City` all at the same time. We'll choose Germany in 1893 because Jonathan went through there first. It will have:

- One country: Germany,
- Three regions: Prussia, Hesse, Saxony
- Two cities for each region, for six in total: Berlin and Königsberg, Darmstadt and Mainz, Dresden and Leipzig.

Here is the insert:

```edgeql
insert Country {
  name := 'Germany',
  regions := {
    (insert Region {
      name := 'Prussia',
      cities := {
        (insert City {
          name := 'Berlin'
        }),
        (insert City {
          name := 'Königsberg'
        }),
      }
    }),
    (insert Region {
      name := 'Hesse',
      cities := {
        (insert City {
          name := 'Darmstadt'
        }),
        (insert City {
          name := 'Mainz'
        }),
      }
    }),
    (insert Region {
      name := 'Saxony',
      cities := {
        (insert City {
          name := 'Dresden'
        }),
        (insert City {
          name := 'Leipzig'
        }),
      }
    })
  }
};
```

With this nice structure set up, we can do things like select a `Region` and see the cities inside it, plus the country it belongs to. And to get `Country` from `Region` we can just use a computed backlink:

```edgeql
select Region {
  name,
  cities: {
    name
  },
  country := .<regions[is Country] {
    name
  }
};
```

With the backlink at the end we have another link between `Country` and its property `regions`, but now we get the `Country` as the output instead. Here's the output:

```
{
  default::Region {
    name: 'Prussia',
    cities: {default::City {name: 'Berlin'}, default::City {name: 'Königsberg'}},
    country: {default::Country {name: 'Germany'}},
  },
  default::Region {
    name: 'Hesse',
    cities: {default::City {name: 'Darmstadt'}, default::City {name: 'Mainz'}},
    country: {default::Country {name: 'Germany'}},
  },
  default::Region {
    name: 'Saxony',
    cities: {default::City {name: 'Dresden'}, default::City {name: 'Leipzig'}},
    country: {default::Country {name: 'Germany'}},
  },
}
```

[Here is all our code so far up to Chapter 19.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you display all the `City` names and the names of the `Region` they are in?

2. How about the `City` names plus the names of the `Region` and the name of the `Country` they are in?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _The race against time._
