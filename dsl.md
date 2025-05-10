# DSL

- 给自己快速又较为精确的表达
- 给 AI 阅读来生成相应的代码

## event/state

采用 algebraic data type 来表示事件和状态（括号里面表示的是参数）

```
event
  | stated(duration)
  | paused 
  | resumed 
  | reset 
  | _ticked(duration)
```

对应的dart语法

```dart
sealed class Event {
  const Event();
}

final class Stated extends Event {
  final Duration duration;
  const Stated(this.duration);
}

final class Paused extends Event {
  const Paused();
}

final class Resumed extends Event {
  const Resumed();
}

final class Reset extends Event {
  const Reset();
}

final class _Ticked extends Event {
  final Duration duration;
  const _Ticked(this.duration);
}
```

## bloc

尽量一行来表达分支意思，都要是state，然后副作用用 & 标注

```
bloc
  | started -> runInProgress & _ticker.periodic(_ticked)
  | paused  -> switch(state)
                 | runInProgress  -> runPause & _ticker.pause()
                 | _              -> state
  | resumed -> switch(state)
                 | runPause       -> runInProgress & _ticker.resume()
                 | _              -> state
  | reset   -> initial & _ticker.cancel()
  | _ticked -> duration > 0 
                 ? RunInProgress
                 : RunComplete
```

对应的dart语法

```dart
class TimerBloc extends Bloc<Event, State> {
  final Ticker _ticker;
  late StreamSubscription<Duration> _tickerSubscription;

  TimerBloc({required Ticker ticker})
      : _ticker = ticker,
        super(const Initial()) {
    on<Started>(_onStarted);
    on<Paused>(_onPaused);
    on<Resumed>(_onResumed);
    on<Reset>(_onReset);
    on<_Ticked>(_onTicked);
  }

  void _onStarted(Started event, Emitter emit) {
    emit(const RunInProgress(0));
    _tickerSubscription = _ticker.tick().listen((duration) {
      add(_Ticked(duration));
    });
  }

  void _onPaused(Paused event, Emitter emit) {
    if (state is RunInProgress) {
      _tickerSubscription.pause();
      emit(RunPause(state.duration));
    }
  }

  void _onResumed(Resumed event, Emitter emit) {
    if (state is RunPause) {
      _tickerSubscription.resume();
      emit(RunInProgress(state.duration));
    }
  }

  void _onReset(Reset event, Emitter emit) {
    _tickerSubscription.cancel();
    emit(const Initial());
  }

  void _onTicked(_Ticked event, Emitter emit) {
    final duration = state.duration - event.duration;
    if (duration > Duration.zero) {
      emit(RunInProgress(duration));
    } else {
      emit(const RunComplete());
    }
  }
}
```

## 测试

测试目的+操作，可以用中文表达，但是代码里面用英文

### 测试 state/event

```
测试等价性：
  TimerInitial(60) == TimerInitial(60)
```

dart语法

```dart
test('TimerInitial equality', () {
  expect(TimerInitial(60), TimerInitial(60));
});
```

### 测试 bloc

一般测试的是初始状态，测试输入事件后输出的状态

- 初始状态

  ```
  初始状态
    state == TimerInitial(60)
  ```

  dart语法

  ```dart
  test('Initial state is TimerInitial', () {
    expect(bloc.state, const TimerInitial(60));
  });
  ```

- 输入输出

  seedState --event--> expectState

  ```
  5 s的时候 resume 状态为 RunInProgress(5)
    RunPause(5) --resumed--> RunInProgress(5)
  ```

  dart语法

  ```dart
  blocTest<TimerBloc, TimerState>(
    'emits [TickerRunInProgress(5)] when ticker is resumed at 5',
    build: () => TimerBloc(ticker: ticker),
    seed: () => TimerRunPause(5),
    act: (bloc) => bloc.add(TimerResumed()),
    expect: () => [TimerRunInProgress(5)],
  );
  ```

  通过seed来设置当前状态

- 有mock依赖的时候

  ```
  测试 xxxx // 让 AI 自动补全
    mock
      ticker.tick(ticks: 3) => Stream(3)
    _ --Started(3)--> RunInProgress(3)
    called 1 ticker.tick(ticks: 3)
  ```

  - mock 缩进表示进行 mock 代理
  - _ 表示初始状态
  - called 1 xxx 表示检查函数调用

  dart 语法

  ```dart
  blocTest<TimerBloc, TimerState>(
    'emits [TimerRunInProgress(3)] when timer ticks to 3',
    setUp: () {
      when(() => ticker.tick(ticks: 3)).thenAnswer(
        (_) => Stream<int>.value(3),
      );
    },
    build: () => TimerBloc(ticker: ticker),
    act: (bloc) => bloc.add(TimerStarted(duration: 3)),
    expect: () => [TimerRunInProgress(3)],
    verify: (_) => verify(() => ticker.tick(ticks: 3)).called(1),
  );
  ```

### 测试 UI

一般都是需要 mock 的，因为需要依赖 bloc

- 测试渲染组件：

  ```
  存在暂停和重置按钮
    mock
      timerBloc.state => RunInProgress(59)
    pumpTimer
    find.text('00:59')
    find.byIcon(Icons.pause)
    find.byIcon(Icons.replay)
  ```

  dart 语法

  ```dart
  testWidgets('renders timer with pause and reset buttons', (tester) async {
    when(() => timerBloc.state).thenReturn(const TimerRunInProgress(59));

    await tester.pumpWidget(
      BlocProvider.value(
        value: timerBloc,
        child: const TimerView(),
      ),
    );

    expect(find.text('00:59'), findsOneWidget);
    expect(find.byIcon(Icons.pause), findsOneWidget);
    expect(find.byIcon(Icons.replay), findsOneWidget);
  });
  ```

- 测试 action 组件

  需要验证对bloc的调用

  ```
  replay按钮被点击
    mock
      timerBloc.state => RunInProgress(30)
    pumpTimer
    tap(find.byIcon(Icons.replay))
    verify(() => timerBloc.add(const TimerReset())).called(1);
  ```

  dart 语法

  ```dart
  testWidgets('replay button calls TimerReset event', (tester) async {
    when(() => timerBloc.state).thenReturn(const TimerRunInProgress(30));

    await tester.pumpWidget(
      BlocProvider.value(
        value: timerBloc,
        child: const TimerView(),
      ),
    );

    await tester.tap(find.byIcon(Icons.replay));
    verify(() => timerBloc.add(const TimerReset())).called(1);
  });
  ```