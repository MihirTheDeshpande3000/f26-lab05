# reservation-service: Smells and One Fix

## Milestone 1: Three smells

### Smell 1

**The smell.** Duplication over reuse. `ReservationManager` has its own conflict check even though `availability.ts` already has the same overlap rule in `isSlotFree`.

**Classic or agent-specific.** Agent-specific. This looks like missing context: the conflict logic was written again in `ReservationManager.hasConflict` instead of reusing the helper that already does it.

**Where in the code.** `src/reservationManager.ts`, in `ReservationManager.createBooking` and the private `hasConflict` method. The duplicated rule is in `src/availability.ts`, in `isSlotFree`.

**The principle it violates.** Information hiding / localizing change. There should be one place that owns the rule for whether two time ranges overlap.

**What it makes expensive.** If the overlap rule ever changes, both `hasConflict` and `isSlotFree` have to be changed. If one gets missed, booking creation could say a slot conflicts while the availability code says the same slot is free, or vice versa.

### Smell 2

**The smell.** Speculative over-abstraction. The notification code has a registry and factory setup even though email is the only notification channel that exists.

**Classic or agent-specific.** Agent-specific. This looks like an underspecified request: the generated code planned for a bunch of possible notification channels before there was actually a second one to support.

**Where in the code.** `src/notifications/notifierFactory.ts`, especially `ChannelName`, the `builders` map, `registerChannel`, `registeredChannels`, and `createNotificationChannel`.

**The principle it violates.** Decouple what actually varies. Right now there is nothing to select between because `ChannelName` can only be `'email'`, so the extra abstraction is solving a problem the program does not have yet.

**What it makes expensive.** Even a normal change to how email notifications are created now means understanding the registry, builder type, factory, and config path. That is extra code and indirection for a system that still only has one implementation.

### Smell 3

**The smell.** Phantom complexity. The service has a configurable TTL cache and checks it when listing bookings for a room, but `ReservationManager` never puts anything into that cache.

**Classic or agent-specific.** Agent-specific. This looks like free volume: a full cache abstraction was generated, including expiration and configuration, without the basic get/set path ever being completed in the manager.

**Where in the code.** `src/cache/queryCache.ts`, `src/cache/cacheConfig.ts`, and `ReservationManager.listBookingsForRoom` in `src/reservationManager.ts`.

**The principle it violates.** Keep complexity tied to an actual requirement. The cache adds another piece of the design to understand even though it currently does not help the room-booking query at all.

**What it makes expensive.** Anyone changing room queries has to think about TTLs, stale data, invalidation, and cache configuration even though nothing is actually being cached. If the cache starts getting used later, booking creation and cancellation also have to invalidate it correctly or the service can return old booking data.

---

## Milestone 2: One small fix

**Which smell you attacked.** Smell 1, duplication over reuse. There is already a clear owner for the overlap rule in `availability.ts`, so this is also the smallest of the three smells to fix without changing behavior.

**What changed.** In `src/reservationManager.ts`, I imported `isSlotFree` from `availability.ts`. `createBooking` now checks `!isSlotFree(request.start, request.end, existing.start, existing.end)` instead of calling `hasConflict`. I then removed the private `hasConflict` method. The actual overlap behavior stays the same, but the rule now only exists in one place.

**What you deliberately did not touch.** I only changed the duplicated overlap check. I did not restructure `createBooking`, change how times are represented, rewrite `findFreeSlots`, or clean up other duplication like the pricing logic. None of that is needed to fix this smell, and doing it would make the change much bigger than it needs to be.

**How you know behavior is preserved.** I will run `npm test` and `npm run typecheck` after the change and make sure both are green without editing any tests. The booking tests already cover full overlaps, partial overlaps, bookings that touch at an endpoint, different rooms, and using a time again after cancellation. The availability tests also check that occupied slots are removed while slots that only touch another booking stay available. That gives good coverage of the exact rule I changed. It would not catch every possible pair of times or tell me whether the intended overlap policy itself should change later.

---

## Milestone 3: Two proposals and one false positive

### Proposal A (not coded)

**The problem.** Speculative over-abstraction in `src/notifications/notifierFactory.ts`. The code has a registry/factory system for notification channels, but the only channel it can actually build is email.

**The decomposition.** I would keep `NotificationChannel` as the small interface that reservation code talks to, and let `EmailChannel` own the email-specific work. I would remove the builder registry, `registerChannel`, and `registeredChannels` while email is the only implementation. `ReservationManager` can just receive a `NotificationChannel`, with an `EmailChannel` created during setup. If another real channel gets added later, then I would add whatever selection logic that actual requirement needs instead of guessing at it now.

**One cost.** This means adding a second channel later would take a little more work because there would not already be a runtime registration system waiting for it. I would still prefer the simpler version while email is the only channel, because right now the registry is extra machinery with nothing to vary. If the service later had several channels that had to be chosen dynamically from configuration, then I would keep or bring back a factory/registry because at that point it would be solving a real problem.

### Proposal B (not coded)

**The problem.** Phantom complexity in the query cache. `ReservationManager` creates and reads from a `QueryCache`, but it never stores the result of `findByRoom` in it, so the cache cannot actually speed up that query.

**The decomposition.** For the current system, I would remove `QueryCache` from `ReservationManager` and have room-booking queries go straight through `StorageProvider`. `ReservationManager` would own the booking workflow and `StorageProvider` would own getting and storing booking data. If performance later shows that these queries really need caching, I would put the cache around the storage/query boundary so one component owns both filling the cache and invalidating it.

**One cost.** Removing it means there is no cache already sitting there if room queries become slow later, so caching would have to be added back and tested then. I would still prefer removing it now because there is no current benefit to paying the complexity cost. If profiling later showed that `findByRoom` was actually a bottleneck, then I would choose the cached design instead, but I would add it at the storage/query boundary rather than making `ReservationManager` manage cache rules itself.

### The thing that looks smelly but is fine

**What it is.** `src/reservationManager.ts`, `ReservationManager.createBooking`. At first it can look like a long-method smell because several steps happen in one method.

**Why it is fine.** Most of the method is just coordinating one operation: create a booking. Validation is handled by `validateReservationRequest`, storage goes through `StorageProvider`, pricing is pushed into `calculatePrice`, and notification work goes through `dispatchNotification` / `NotificationChannel`. So `createBooking` is mostly saying what order those steps happen in instead of implementing every one of them itself.

**What would flip your verdict.** I would call it a real problem if more detailed logic started getting shoved directly into `createBooking`, like multiple pricing policies, email formatting, storage-specific code, or several different booking modes. At that point it would stop being a coordinator for one use case and start owning a bunch of separate things that change for different reasons.
