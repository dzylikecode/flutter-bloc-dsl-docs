
## 构建 mock 对象

普通对象

```dart
class MockAuthenticationRepository extends Mock
    implements AuthenticationRepository {}
```

bloc 层级对象（一般用在ui当中）

```dart
class _MockCounterCubit extends MockCubit<int> implements CounterCubit {}


class _MockTimerBloc extends MockBloc<TimerEvent, TimerState>
    implements TimerBloc {}
```


## 代理返回值

普通值

```dart
when(() => timerBloc.state).thenReturn(const TimerInitial(60));
```


异步值

```dart
when(() => mockAuthenticationRepository.isSignedIn())
    .thenAnswer((_) async => true);
```

函数返回值

```dart
when(() => counterCubit.increment()).thenReturn(null);
```

## 检测函数调用

```dart
verify(() => counterCubit.decrement()).called(1);
```

测试含有函数以及参数

```dart
verify(() => ticker.tick(ticks: 5)).called(1)

// 参数不是 equalable
verify(
  () => timerBloc.add(
    any(
      that: isA<TimerStarted>().having((e) => e.duration, 'duration', 60),
    ),
  ),
).called(1);
```