---
title: Code review of an Order object implemented as a state machine 
created_at: 2021-03-20T212:42:46.712Z
author: Andrzej Krzywda
tags: [rails event store, ddd, aggregate, event sourcing]
publish: false
---

Let me show you an example piece of code - an Order class. This class is used in our sample DDD/CQRS/ES application. We're in the process of improving this app so this is a good opportunity to document certain opinions and changes.

```ruby
module Ordering
  class Order
    include AggregateRoot

    AlreadySubmitted = Class.new(StandardError)
    AlreadyPaid = Class.new(StandardError)
    NotSubmitted = Class.new(StandardError)
    OrderHasExpired = Class.new(StandardError)
    MissingCustomer = Class.new(StandardError)

    def initialize(id)
      @id = id
      @state = :draft
    end

    def submit(order_number, customer_id)
      raise AlreadySubmitted if @state.equal?(:submitted)
      raise OrderHasExpired  if @state.equal?(:expired)
      raise MissingCustomer unless customer_id
      apply OrderSubmitted.new(data: {order_id: @id, order_number: order_number, customer_id: customer_id})
    end

    def confirm(transaction_id)
      raise OrderHasExpired if @state.equal?(:expired)
      raise NotSubmitted unless @state.equal?(:submitted)
      apply OrderPaid.new(data: {order_id: @id, transaction_id: transaction_id})
    end

    def expire
      raise AlreadyPaid if @state.equal?(:paid)
      apply OrderExpired.new(data: {order_id: @id})
    end

    def add_item(product_id)
      raise AlreadySubmitted unless @state.equal?(:draft)
      apply ItemAddedToBasket.new(data: {order_id: @id, product_id: product_id})
    end

    def remove_item(product_id)
      raise AlreadySubmitted unless @state.equal?(:draft)
      apply ItemRemovedFromBasket.new(data: {order_id: @id, product_id: product_id})
    end

    def cancel
      raise OrderHasExpired if @state.equal?(:expired)
      raise NotSubmitted unless @state.equal?(:submitted)
      apply OrderCancelled.new(data: {order_id: @id})
    end

    on OrderSubmitted do |event|
      @customer_id = event.data[:customer_id]
      @number = event.data[:order_number]
      @state = :submitted
    end

    on OrderPaid do |event|
      @state = :paid
    end

    on OrderExpired do |event|
      @state = :expired
    end

    on OrderCancelled do |event|
      @state = :cancelled
    end

    on ItemAddedToBasket do |event|
    end

    on ItemRemovedFromBasket do |event|
    end
  end
end
```

As always, this class was nice and simple at the beginning, but over time it grew and became less readable.

There are now several responsibilities of this class. Some of those I will leave for another discussion (like coupling the domain code with event code). Today I want to focus on the concept of a state machine.

This Order object has now 5 possible states. How come it grew to such size? As always - one by one. 5 is probably still OKish but we can imagine what can happen if we extend it even more.

It's worth noting that this class uses AggregateRoot which helps in the event sourcing part of this object.

What are the requirements for this state machine?

More or less this:


| Order     | draft | submitted | paid  | expired  | cancelled |
|-----------|:-----:|:---------:|:-----:|:--------:|:---------:|
| draft     |       |     ✅    |       |   ✅      |         |
| submitted |       |           |   ✅  |          |   ✅     |
| paid      |       |           |       |          |           |
| expired   |       |           |       |          |           |
| cancelled |       |           |       |          |           |

Draft is the initial state. The happy path then switches to `submitted`, then to `paid`. 
The less happy paths include `expired` and `cancelled`, both are leaf states.

The challenge with state machines is that it's not easy to represent them in code in a readable manner. Whenever the number of states and transitions grows it's becoming harder to read such code.

State machines consist of states and transitions. Somehow we need to represent them in the code. In this implementation, we put the transition as the main "dimension". The method names show the possible transitions. However, they show possible transitions for all the states. This leads to a problem, that in each of those methods we now need to "disable" the impossible transitions. We could do it with just an early return in such cases without using exceptions.

Exactly, why do we use exceptions here?

Because one responsibility of this object is to communicate "WHY" a certain change is not possible. An early return only communicates a boolean information - possible or not. A custom exception brings more context.
