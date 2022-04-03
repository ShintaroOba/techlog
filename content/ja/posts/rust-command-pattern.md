---
type: Archive
title: 「Rust Design Pattern」を翻訳してみた(1) ～Commandパターン編～
date: 2022-04-01
description: Commandパターン
titleWrap: wrap
tags: 
- 翻訳
- rust

image: images/summary/command.png
---

# Introduction
Rustaceanになるには、読むだけじゃなくてとにかく手を動かしてコードを書いてみなくては。  
Rustのイディオムやデザインパターンから雰囲気だけでも感じ取ろうと読破を目標に読んでいくシリーズ。ソースコードは本家からのコピーではなく、書きおろしなので細かい設定が甘いところが散見されると思いますが、ご容赦ください。

元の文章はこちら。  
{{< blogcard url="https://rust-unofficial.github.io/patterns/patterns/behavioural/command.html" >}}  

# Commandパターン
Commandパターンの基本の考えは、振舞いを独自のオブジェクトに分割し、それをパラメータとして外部から渡すこと。

## 使い時
- オブジェクトにカプセル化された処理をシーケンシャルに実行したい場合
- 処理が追加される可能性がある場合
- 処理の履歴を残したい場合



## 実例
ダイエットプログラムを想定。ダイエットをするために、いくつかのダイエットプログラムを構築し、それを順番に実行していくプログラムを作成する。


まずは、以下のDietTraitを作成。「走る」と「ワークアウト」の2つの振舞いを持ち、それぞれ消費したカロリーを返す。


````rs
pub trait Diet {
    fn run(&self) -> Calorie;
    fn workout(&self) -> Calorie;
}
````

カロリーはi16の整数型をラップしたユニット構造体として定義し、
カロリーの和を計算するplusメソッドを用意。

````rs
#[derive(Debug)]
pub struct Calorie(i16);

impl Calorie {
    fn plus(&self, cal: &Calorie) -> Calorie {
        Calorie(&self.0 + cal.0)
    }
}

````

今回利用するダイエットプログラムとして、有名な2社のフィットネス企業のプログラムを利用可能とする。2社の構造体をDietTraitを継承する形で定義する。
Rizapはワークアウトの消費カロリーが多く、Tipnessはランニングの消費カロリーが多い。

````rs

pub struct Rizap;
impl Diet for Rizap {
    fn run(&self) -> Calorie {
        Calorie(700)
    }
    fn workout(&self) -> Calorie {
        Calorie(600)
    }
}

pub struct Tipness;
impl Diet for Tipness {
    fn run(&self) -> Calorie {
        Calorie(800)
    }
    fn workout(&self) -> Calorie {
        Calorie(400)
    }
}
````

次に、今回のCommandPatternで肝となるのが、下記のActiveProgram。
ダイエット希望者は、ActiveProgram構造体のcommandsに任意のダイエットプログラムを追加していくことで、自分だけのダイエットプログラムを組み立てることができる。
(commandsはVec<Box<dyn Diet>>型であるため、Dietを継承したRizapやTipnessを任意の数追加することができる。)  
run, workoutメソッドは、ActiveProgram自身のcommand変数内部の命令句をイテレーションして実行する。


````rs
pub struct ActiveProgram {
    commands: Vec<Box<dyn Diet>>,
}

impl ActiveProgram {
    fn new_now() -> Self {
        Self { commands: vec![] }
    }

    // cmdを追加することができるメソッド
    fn add_diet(&mut self, cmd: Box<dyn Diet>) {
        self.commands.push(cmd)
    }

    fn run(&self) -> Vec<Calorie> {
        self.commands.iter().map(|cmd| cmd.run()).collect()
    }
    fn workout(&self) -> Vec<Calorie> {
        self.commands.iter().map(|cmd| cmd.workout()).collect()
    }
}

````

全然必須ではないのだが、今回はパーソナルプログラム感を出したかったので、
Personオブジェクトを作成し、この構造体にdiet_programへの命令追加や、命令実行操作をラップするメソッドを作成。
クライアントとなる``main()``からはPersonインスタンスを通して命令の追加、実行を行う。
````rs
pub struct Person {
    pub diet_program: ActiveProgram,
}
impl Person {
    fn add_program(&mut self, cmd: Box<dyn Diet>) {
        self.diet_program.add_diet(cmd)
    }
    fn start_workout(&self) -> Vec<Calorie> {
        self.diet_program.workout()
    }
    fn start_run(&self) -> Vec<Calorie> {
        self.diet_program.run()
    }
}

````

最後にMain関数が下記。カロリーの総和を出す部分はクライアント側ではなく、内部処理で隠蔽したい気持ちもあるが、今回はここまでで...
処理が追加される場合、何らかのトリガー処理によって命令が実行される場合などに活用できそうな感じは伝わったと思います。

````rs
fn main() {
    let mut john = Person {
        diet_program: ActiveProgram::new_now(),
    };

    john.add_program(Box::new(Rizap));
    john.add_program(Box::new(Tipness));

    let mut workout_cal = Calorie(0);
    john.start_workout()
        .iter()
        .for_each(|cal| workout_cal = workout_cal.plus(cal));

    println!("workout calorie is {:?}", workout_cal);
    assert_eq!(1000, workout_cal.0);
    
    
    let mut run_cal = Calorie(0);
    john.start_run()
        .iter()
        .for_each(|cal| run_cal = run_cal.plus(cal));

    println!("run calorie is {:?}", run_cal);
    assert_eq!(1500, run_cal.0);
}

````

最後に全体のソースを貼り付ける。
````rs
pub trait Diet {
    fn run(&self) -> Calorie;
    fn workout(&self) -> Calorie;
}
#[derive(Debug)]
pub struct Calorie(i16);

impl Calorie {
    fn plus(&self, cal: &Calorie) -> Calorie {
        Calorie(&self.0 + cal.0)
    }
}

pub struct ActiveProgram {
    commands: Vec<Box<dyn Diet>>,
}
impl ActiveProgram {
    fn new_now() -> Self {
        Self { commands: vec![] }
    }

    fn add_diet(&mut self, cmd: Box<dyn Diet>) {
        self.commands.push(cmd)
    }

    fn run(&self) -> Vec<Calorie> {
        self.commands.iter().map(|cmd| cmd.run()).collect()
    }
    fn workout(&self) -> Vec<Calorie> {
        self.commands.iter().map(|cmd| cmd.workout()).collect()
    }
}

pub struct Rizap;
impl Diet for Rizap {
    fn run(&self) -> Calorie {
        Calorie(700)
    }
    fn workout(&self) -> Calorie {
        Calorie(600)
    }
}

pub struct Tipness;
impl Diet for Tipness {
    fn run(&self) -> Calorie {
        Calorie(800)
    }
    fn workout(&self) -> Calorie {
        Calorie(400)
    }
}
pub struct Person {
    pub diet_program: ActiveProgram,
}
impl Person {
    fn add_program(&mut self, cmd: Box<dyn Diet>) {
        self.diet_program.add_diet(cmd)
    }
    fn start_workout(&self) -> Vec<Calorie> {
        self.diet_program.workout()
    }
    fn start_run(&self) -> Vec<Calorie> {
        self.diet_program.run()
    }
}

fn main() {
    let mut john = Person {
        diet_program: ActiveProgram::new_now(),
    };

    john.add_program(Box::new(Rizap));
    john.add_program(Box::new(Tipness));

    let mut workout_cal = Calorie(0);
    john.start_workout()
        .iter()
        .for_each(|cal| workout_cal = workout_cal.plus(cal));

    println!("workout calorie is {:?}", workout_cal);
    assert_eq!(1000, workout_cal.0);
    let mut run_cal = Calorie(0);
    john.start_run()
        .iter()
        .for_each(|cal| run_cal = run_cal.plus(cal));

    println!("run calorie is {:?}", run_cal);
    assert_eq!(1500, run_cal.0);
}


````