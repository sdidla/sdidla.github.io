---
layout: post
title:  "Asynchronous reduce"
date:   2021-05-06 14:20:28 +0200
categories: welcome
---

A publisher that performs an asynchronous reduce operation and outputs `ReduceEvent` downstream

```swift
public enum ReduceEvent<State, Action> {
    case willChangeState(_ state: State, action: Action)
    case didChangeState(_ state: State, action: Action)
}

extension Publisher {

    func serialReduce<State, Action, P>(_ initialState: State, transform: @escaping (State, Action) ->  P?)
    -> AnyPublisher<ReduceEvent<State, Action>, Never>
    where P : Publisher, P.Output == State, P.Failure == Never, Output == Action, Failure == Never {
        
        reduceToEventPublisher(initialState, transform: transform)
            .buffer(size: .max, prefetch: .keepFull, whenFull: .dropOldest)
            .flatMap(maxPublishers: .max(1)) { $0 }
            .share()
            .eraseToAnyPublisher()
    }

    func concurrentReduce<State, Action, P>(_ initialState: State, transform: @escaping (State, Action) ->  P?)
    -> AnyPublisher<ReduceEvent<State, Action>, Never>
    where P : Publisher, P.Output == State, P.Failure == Never, Output == Action, Failure == Never {
        
        reduceToEventPublisher(initialState, transform: transform)
            .flatMap { $0 }
            .share()
            .eraseToAnyPublisher()
    }

    func latestReduce<State, Action, P>(_ initialState: State, transform: @escaping (State, Action) ->  P?)
    -> AnyPublisher<ReduceEvent<State, Action>, Never>
    where P : Publisher, P.Output == State, P.Failure == Never, Output == Action, Failure == Never {
        
        reduceToEventPublisher(initialState, transform: transform)
            .switchToLatest()
            .share()
            .eraseToAnyPublisher()
    }


    private func reduceToEventPublisher<State, Action, P>(_ initialState: State, transform: @escaping (State, Action) ->  P?)
    -> AnyPublisher<AnyPublisher<ReduceEvent<State, Action>, Never>,Never>
    
    where P : Publisher, P.Output == State, P.Failure == Never, Output == Action, Failure == Never {
        
        var currentState = initialState
        
        return map { action in
            Deferred {
                Just(transform)
                    .compactMap { $0(currentState, action) }
                    .flatMap { newPublisher in
                        newPublisher
                            .handleEvents(receiveOutput: { currentState = $0 })
                            .map { .didChangeState($0, action: action) }
                            .prepend(.willChangeState(currentState, action: action))
                    }
            }
            .eraseToAnyPublisher()
        }
        .eraseToAnyPublisher()
    }
}
```