```java

import java.util.*;
import java.util.concurrent.*;

class Snake extends Mover{
    public Snake(int start,int end){
        super(start,end);
    }
}
class Ladder extends Mover{
    public Ladder(int start,int end){
        super(start,end);
    }
}
abstract class Mover{
    public int start;
    public int end;
    public Mover(int start, int end){
        this.start = start;
        this.end = end;
    }
    public int move(){
        return end;
    }
}
class Dice{
    int Roll(){
        return (int)(Math.random()*6)+1;
    }
}
class Player{
    int id;
    int position;
    int getPosition(){ return position;}
    void setPosition(int newPosition){ this.position = newPosition}
}

class Board{
    Map<Integer,Mover> moverMap = new HashMap<>();
}

class TurnResponse{
    int diceRollNum;
    boolean moved;
    public TurnResponse(int diceRollNum, boolean moved){
        this.diceRollNum = diceRollNum;
        this.moved = moved;
    }
}

class BoardController{
    Board board;
    List<Player> players;
    Dice dice;
    public final ExecutorService executorService = Executors.newSingleThreadExecutor();

    void startGame() throws ExecutionException, InterruptedException {
        Deque<Player> playerQueue = new LinkedList<>(players);
        while(true){
            Player currentPlayer = playerQueue.poll();

            try {
                if (currentPlayer == null) {
                    System.out.println("Invalid Player");
                    return;
                }
                CompletableFuture<TurnResponse> res = CompletableFuture.supplyAsync(() -> takeTurn(currentPlayer), executorService).orTimeout(10, TimeUnit.SECONDS);
                TurnResponse turnResponse = res.get();
                if (currentPlayer.getPosition() == 100) {
                    return;
                } else {
                    if (turnResponse.diceRollNum == 6 && turnResponse.moved) {
                        playerQueue.addFirst(currentPlayer);
                    } else {
                        playerQueue.addLast(currentPlayer);
                    }
                }
            } catch (ExecutionException e) {
                if(e.getCause() instanceof TimeoutException){
                    System.out.println("Removing current player from game due to timeout: "+currentPlayer.id);
                }
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }

    TurnResponse takeTurn(Player player){
        int currentPos = player.getPosition();
        int diceNum = dice.Roll();
        int pos = currentPos+diceNum;
        if(currentPos+diceNum>100){
            return new TurnResponse(diceNum,false);
        }
        if(board.moverMap.containsKey(pos)){
            pos = board.moverMap.get(pos).end;
        }
        player.setPosition(pos);
        if(pos==100){
            System.out.println("Player with playerId:"+player.id+"Won");
            System.out.println("Game Ends");
        }
        return new TurnResponse(diceNum,true);
    }
}

```

``` java

public class KeyValueStore{  
    Map<String, Integer> kvStore;  
    List<Transaction> transactions;  
    KeyValueStore(){  
        kvStore = new HashMap<>();  
        transactions = new ArrayList<>();  
    }  
  
    Integer get(String key){  
        if(transactions.isEmpty())  
            return kvStore.get(key);  
        else{  
            for(int i = transactions.size()-1;i>=0;i--){  
                Map<String,Integer> t = transactions.get(i).tMap;  
                if(t.containsKey(key)){  
                    return t.get(key);  
                }  
            }  
        }  
        return null;  
    }  
  
    void set(String key, Integer value){  
        if(transactions.isEmpty())  
            kvStore.put(key, value);  
        else{  
            int sz = transactions.size();  
            transactions.get(sz-1).tMap.put(key, value);  
        }  
    }  
  
    void createTransaction(){  
        transactions.add(new Transaction());  
    }  
  
    void rollback(){  
        if(!transactions.isEmpty()){  
            int sz = transactions.size();  
            transactions.remove(sz-1);  
        }  
    }  
  
    void commit(){  
        if(transactions.isEmpty()) return;  
        else if(transactions.size()==1){  
            kvStore.putAll(transactions.get(0).tMap);  
            transactions.remove(0);  
        }else{  
            int sz = transactions.size();  
            transactions.get(sz-2).tMap.putAll(transactions.get(sz-1).tMap);  
            transactions.remove(sz-1);  
        }  
    }  
}  
  
class Transaction{  
    Map<String, Integer> tMap;  
    Transaction(){  
        tMap = new HashMap<>();  
    }  
}
```