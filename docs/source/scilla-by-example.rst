Scilla by Example
==================


Ping Pong
###################


Crowdfunding
###################


.. code-block:: ocaml
    :linenos:

    (***************************************************)
    (*               Associated library                *)
    (***************************************************)
    library Crowdfunding

    let andb = 
      fun (b : Bool) =>
      fun (c : Bool) =>
        match b with 
        | False => False
        | True  =>
          match c with 
          | False => False
          | True  => True
          end
        end

    let orb = 
      fun (b : Bool) => fun (c : Bool) =>
        match b with 
        | True  => True
        | False =>
          match c with 
          | False => False
          | True  => True
          end
        end

    let negb = fun (b : Bool) => 
      match b with
      | True => False
      | False => True
      end

    let one_msg = 
      fun (msg : Message) => 
        let nil_msg = Nil {Message} in
        Cons {Message} msg nil_msg
        
    let check_update = 
      fun (bs : Map Address Int) =>
      fun (_sender : Address) =>
      fun (_amount : Int) =>
        let c = builtin contains bs _sender in
        match c with 
        | False => 
          let bs1 = builtin put bs _sender _amount in
          Some {Map Address Int} bs1 
        | True  => None {Map Address Int}
        end

    let blk_leq =
      fun (blk1 : BNum) =>
      fun (blk2 : BNum) =>
        let bc1 = builtin blt blk1 blk2 in 
        let bc2 = builtin eq blk1 blk2 in 
        orb bc1 bc2

    let accepted_code = 1
    let missed_deadline_code = 2
    let already_backed_code  = 3
    let not_owner_code  = 4
    let too_early_code  = 5
    let got_funds_code  = 6
    let cannot_get_funds  = 7
    let cannot_reclaim_code = 8
    let reclaimed_code = 9
      
    (***************************************************)
    (*             The contract definition             *)
    (***************************************************)
    contract Crowdfunding

    (*  Parameters *)
    (owner     : Address,
     max_block : BNum,
     goal      : Int)

    (* Mutable fields *)
    field backers : Map Address Int = Emp Address Int
    field funded : Bool = False

    transition Donate ()
      blk <- & BLOCKNUMBER;
      in_time = blk_leq blk max_block;
      match in_time with 
      | True  => 
        bs  <- backers;
        res = check_update bs _sender _amount;
        match res with
        | None => 
          msg  = {_tag : Main; _recipient : _sender; _amount : 0; 
                  code : already_backed_code};
          msgs = one_msg msg;
          send msgs
        | Some bs1 =>
          backers := bs1; 
          accept; 
          msg  = {_tag : Main; _recipient : _sender; _amount : 0; 
                  code : accepted_code};
          msgs = one_msg msg;
          send msgs     
        end  
      | False => 
        msg  = {_tag : Main; _recipient : _sender; _amount : 0; 
                code : missed_dealine_code};
        msgs = one_msg msg;
        send msgs
      end 
    end

    transition GetFunds ()
      is_owner = builtin eq owner _sender;
      match is_owner with
      | False => 
        msg  = {_tag : Main; _recipient : _sender; _amount : 0; 
                code : not_owner_code};
        msgs = one_msg msg;
        send msgs
      | True => 
        blk <- & BLOCKNUMBER;
        in_time = blk_leq blk max_block;
        c1 = negb in_time;
        bal <- balance;
        c2 = builtin lt bal goal;
        c3 = negb c2;
        c4 = andb c1 c3;
        match c4 with 
        | False =>  
          msg  = {_tag : Main; _recipient : _sender; _amount : 0; 
                  code : cannot_get_funds};
          msgs = one_msg msg;
          send msgs
        | True => 
          tt = True;
          funded := tt;
          msg  = {_tag : Main; _recipient : owner; _amount : bal; 
                  code : got_funds_code};
          msgs = one_msg msg;
          send msgs
        end
      end   
    end

    (* transition ClaimBack *)
    transition ClaimBack ()
      blk <- & BLOCKNUMBER;
      after_deadline = builtin blt max_block blk;
      match after_deadline with
      | False =>
        msg  = {_tag : Main; _recipient : _sender; _amount : 0; 
                code : too_early_code};
        msgs = one_msg msg;
        send msgs
      | True =>
        bs <- backers;
        bal <- balance;
        (* Goal has not been reached *)
        f <- funded;
        c1 = builtin lt bal goal;
        c2 = builtin contains bs _sender;
        c3 = negb f;
        c4 = andb c1 c2;
        c5 = andb c3 c4;
        match c5 with
        | False =>
          msg  = {_tag : Main; _recipient : _sender; _amount : 0; 
                  code : cannot_reclaim_code};
          msgs = one_msg msg;
          send msgs
        | True =>
          res = builtin get bs _sender;
          match res with
          | None =>
            msg  = {_tag : Main; _recipient : _sender; _amount : 0; 
                    code : cannot_reclaim_code};
            msgs = one_msg msg;
            send msgs
          | Some v =>
            bs1 = builtin remove bs _sender;
            backers := bs1;
            msg  = {_tag : Main; _recipient : _sender; _amount : v; 
                    code : reclaimed_code};
            msgs = one_msg msg;
            send msgs
          end
        end
      end  
    end


