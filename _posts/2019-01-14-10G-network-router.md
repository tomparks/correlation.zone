---
title:  "10G Network Router in Haskell"
date:   2019-01-14 08:07:40 -07:00
categories: Haskell FPGA
layout: post
---


As part of Noa Z's course, we modified the NetFPGA SUME reference design to support cut-through low latency switching at 10G line rate.

As part of this project a new output port lookup module was written in CλaSH, a Haskell derived DSL compiling to Verilog. The pipelined design was exceptionally succinct when compared to the reference Verilog design.

The document is a literate haskell walk-through of the OPL module, for use as a building block for other pipelined systems. The code cannot be used as-is without the closed NetFPGA SUME codebase and so imports/external references are deliberately omitted with salient details given in the text.

## Cut-through switching

A cut-through ethernet switch has a latency that is not dependant on the length of the packet. Cut-through designs commonly have lower average latency and much lower worst case latency than store-and-forward designs. THe advent of FEC in 40G and above speeds obsoletes cutthrough designs as the FEC code, located at the end of the packet, is required to decode the destination MAC for layer 2 routing.

## The architecture of the OPL

The Xilinx MAC's use the AXI-stream protocol for transmitting data. The MAC's use a 156.25MHz internal clock and this design runs at that rate without gearboxes for simplicity.

### AXI Stream and pipeline depth

A sample of the data on a AXI stream bus is given by the following type:

{% highlight haskell %}
type Tdata = BitVector 64 -- 156.25MHz * 64b = 10g (with preamble overhead)
type Tkeep = BitVector 8
-- tuser defined elsewere
type Tvalid = Bit
type Tready = Bit
type Tlast = Bit

data Stream = Stream { -- AXI stream single bus width sample
   tdata :: Tdata, -- packet data
   tkeep :: Tkeep, -- 8-bit-granularity valid lines
   tuser :: Tuser, -- metadata
   tvalid :: Tvalid, -- global tkeep
   tready :: Tready, -- backpressure signal
   tlast :: Tlast -- metadata for end of burst
} deriving (Show)

-- it is useful to have empty instances to create instances from
emptyStream :: Stream = Stream { tdata=0, tkeep=0, tuser=0, tvalid=0, tready=1, tlast=0 }
{% endhighlight %}

Our OPL module has 8 cycles of latency. 3 clocks are required to get the source MAC. 4 more clocks are needed in order to interleave the 4 ports of access to the MAC->PORT mapping module. A final clock is used to simplify output logic. The source MAC is required in order to act as a _learning_ switch, as we may need to update our MAC->PORT mapping with this source and so the lookup cannot simply begin until we know this.[^1]

### Pipeline declaration

The pipeline type contains the AXI stream connections within the module, as well as all the metadata the OPL has calculated so far. The module is a "conveyor belt" design that moves the data in a `PipelineStage` forward every cycle, modifying the data as it goes.

{% highlight haskell %}
data PipelineStage =  PipelineStage {
    packet :: Stream,
    pktValid :: Bool,
    pktLast :: Bool,
    flitCount :: Unsigned 8,
    dstMac :: Maybe (BitVector 48),
    srcMac :: Maybe (BitVector 48),
    dstPorts :: Maybe (BitVector 8),
    srcPorts :: (BitVector 8),
    packetLen :: (BitVector 16)
}

pipelineDefault = PipelineStage { packet    = emptyStream,
                                  pktValid  = False, -- first flit?
                                  pktLast   = False,
                                  flitCount = 0,
                                  dstMac    = Nothing,
                                  srcMac    = Nothing,
                                  dstPorts  = Nothing,
                                  srcPorts  = 0,
                                  packetLen = 0}

type OPLState = Vec 7 PipelineStage
{% endhighlight %}

This writing style keeps all the data and metadata together. The downside is much wider datapaths in clash than would be required for just buffering the AXI stream for 8 clocks. The clash compiler did not attempt to do any wiring simplification but this should be removed at elaboration time.

The Haskell Maybe type maps exactly to a Verilog type with a additional valid wire. If a Mabye type is exposed, it is easy enough to make combinatorial packers and unpackers that maintain idiomatic styles in both languages.

### Packet routing functions

Due to the designation mapper being a bottleneck in our design, we disabled support for the NetFPGA host ports. We can use maybe to set the DMA designations to zero only if they were valid.

{% highlight haskell %}
-- to avoid blocking the design, we need to never send to DMA
-- all odd bits are the DMA ports
stripDMA :: Maybe (BitVector 8) -> Maybe (BitVector 8)
stripDMA dst = maybe Nothing (\p -> Just (p .&. ($$(bLit "01010101") :: BitVector 8))) dst
{% endhighlight %}

Next we route the packets based upon the metadata in the current pipeline stage. Haskell pattern matching makes this kind of if-else logic clear. We set the destination for packets to the bcast MAC to all ports, otherwise if the MAC was not seen in the lookup table we send to all but the source port. If we have a successful lookup we use that routing.

Without the INLINE declaration, every haskell function generates a new Verilog module. For small functions like this it is simpler to fold all the logic into one module.

{% highlight haskell %}
extractDst :: TCAMreply -> PipelineStage -> Maybe (BitVector 8)
{-# INLINE extractDst #-} -- we use inline to reduce the number of floating modules in verilog
extractDst _ PipelineStage{dstMac = Just 0xffffffffffff}   = Just ($$(bLit "11111111") :: BitVector 8) -- bcast
extractDst TCAMreply{ lut_miss = 1 } state           = Just (complement (srcPorts state))
extractDst TCAMreply{ lut_hit = 1, dst_ports=dst } _ = Just dst
extractDst _ _ = Nothing
{% endhighlight %}

`applyTcamRpy` looks for the results of a lookup in the current pipeline stage if this flit is the first one in a packet. Else, we replicate the destination from the last flit. This ensures that packets are not fragmented.

{% highlight haskell %}
applyTcamRpy :: PipelineStage -> Bit -> TCAMreply -> PipelineStage
{-# INLINE applyTcamRpy #-} -- we use inline to reduce the number of floating modules in verilog
applyTcamRpy now rpyenb iTCAM = now { dstPorts = if headFlit now && btb rpyenb
                                                 then stripDMA (extractDst iTCAM now) -- add the new dst
                                                 else dstPorts now } -- no rpy, keep looping
{% endhighlight %}

### The lookup pipeline

The final function is also the largest. At 100 lines of Haskell, it declares the action on each pipeline stage. The type is

{% highlight haskell %}
opl_pass_mealy :: OPLState -> (Stream, TCAMreply, Bit, Bit) -> (OPLState, (Stream, TCAMrequest))
opl_pass_mealy state (iAXI, iTCAM, reqenb, rpyenb) = (s0 ++ s1 ++ s2 ++ s3 ++ s4 ++ s5 ++ s6, (oAXI, oTCAM))
 where
{% endhighlight %}

The body is a large `where` block defining the next values of pipeline stages `s0`-`s6`, and the connection to the later modules.

Our first act is to update the metadata in the pipeline as soon as flits enter. We need to segment packets and extract the dstMAC and part of the srcMAC depending on the current flit number.

{% highlight haskell %}
s0 =
  let
   valid = tkeep iAXI > 0 -- there is valid data in this.
   flitn = if not valid then 0 -- invalid, so not a flit
           else if btb (tlast (packet (state !! 0))) then 1 -- last packet was the end, so this is a new packet
           else (flitCount (state !! 0) + 1) -- valid, so incr the count. 1 is the head
   -- is_this_last = False -- we know that from tlast, added by the 10G port
   -- first 6 octets
   dstmac = if flitn == 1 then slice d47 d0 (tdata iAXI) else (fromJust (dstMac (state !! 0)))
   srcmacPartial = if flitn == 1 then
                      slice d63 d48 (tdata iAXI)
                   else
                      slice d15 d0 (fromJust (srcMac (state !! 0)))
   srcPorts = slice d23 d16 (tuser iAXI)
  in singleton pipelineDefault { packet = iAXI,
                                 pktValid = valid,
                                 flitCount = flitn,
                                 dstMac    = Just dstmac,
                                 srcMac    = Just ( (0 :: BitVector 32) ++# srcmacPartial ),
srcPorts = srcPorts }
{% endhighlight %}

By stage 2 we know the full srcMAC and can prepare to make a lookup request. If the packet uses etype for size, we trust  this. Otherwise we fall back on a possible prior module to have filled it in. Failing that we set the max length in wide use. This also shows the general form of performing a calculation only if this is the first flit in a packet as determined earlier.

{% highlight haskell %}
-- stage 2: add src mac, - r0
-- if headFlit, then packet that just came in is flit 2 - has src and len
s1 = if headFlit (state !! 0) then
       singleton (state !! 0) { srcMac = Just ( (slice d31 d0 (tdata iAXI)) ++#
                                                (slice d15 d0 (fromJust (srcMac (state !! 0))))
                                         ),
                                packetLen =  let
                                                etype = slice d31 d16 (tdata iAXI)
                                             in
                                                -- etype was protocol indicator, we can't determine the length of frame without store/forward
                                                if etype <= 1500 then etype
                                                else if slice d15 d0 (tuser iAXI) > 0 then slice d15 d0 (tuser iAXI)
                                                else 9000 -- max length
                              }
     else
singleton (state !! 0) { srcMac = srcMac (state !! 1), packetLen = packetLen (state !! 1) }
{% endhighlight %}

The next stages are identical. A outside module controls which of the 4 ports the reply from the mapping module affects, and so we repeat this block 4 times to ensure we catch the reply.

{% highlight haskell %}
s2 = singleton (applyTcamRpy (state !! 1) rpyenb iTCAM)
s3 = singleton (applyTcamRpy (state !! 2) rpyenb iTCAM)
s4 = singleton (applyTcamRpy (state !! 3) rpyenb iTCAM)
s5 = singleton (applyTcamRpy (state !! 4) rpyenb iTCAM)
{% endhighlight %}

Finally we put all our metadata into the TUSER part of the pipeline.

{% highlight haskell %}
s6 = let -- add the output port and other tuser stuff
       s5now = (state !! 5)
       s6now = (state !! 6)
     in let
       newtuser = if headFlit s5now then
                    (slice d127 d32 (tuser (packet s5now))) ++#
                    (fromJust (dstPorts s5now)) ++#
                    (srcPorts  s5now) ++#
                    (packetLen s5now)
                  else if (pktValid s5now) then -- loop it
                    tuser (packet s6now) -- prev packet
                  else
                    (0 :: Tuser)
     in  
singleton s5now { packet = (packet s5now) { tuser = newtuser }}

oAXI = (packet (state !! 6)) -- cycle after assignment to s6 for any given flit
{% endhighlight %}

The only remaining logic is to use an outside signal to send mapping lookup requests at the right time.

{% highlight haskell %}
oTCAM =
  let
    -- if pkt in fst pipeline stage is flitcount elem [1, 2, 3, 4] then we have the head, this is idx
    headloc = if flitCount (state !! 0) >= 2 && flitCount (state !! 0) <= 5 then
      Just ((flitCount (state !! 0)) - 1)
      else Nothing
  in
    if isJust headloc && btb reqenb
         then TCAMrequest { lookup_req = 1,
                            dst_mac = fromJust (dstMac (state !! (fromJust headloc))),
                            src_mac = fromJust (srcMac (state !! (fromJust headloc))),
                            src_port = (srcPorts (state !! (fromJust headloc))) }
else tcam_r_default
{% endhighlight %}

And whist fairly terse that is the full lookup logic for a 10g 4 port L2 switch. The benefits of maintaining a clear state machine may be felt even more when attempting to do kinds of low latency L3 switching.

### Outside module declaration

We define the initial state of the module as a empty pipeline (this also allows a reset wire to fully reset the module).
Next we use a ANNotation to subdivide the ports into the logic we need for outside interfacing, declare a clock and a reset that may be async, and mark this function as the compile target by assigning to `topEntity`.

{% highlight haskell %}
oplInitalState :: OPLState = replicate (SNat :: SNat 7) pipelineDefault

{-# ANN topEntity
  (Synthesize
    { t_name     = "OPL"
    , t_inputs   = [
        PortName "clk",
        PortName "rst",
        PortProduct "" [
                        PortProduct "" [PortName "I_DATA", PortName "I_KEEP", PortName "I_USER",
                                        PortName "I_VALID", PortName "I_READY", PortName "I_LAST"],
                        PortProduct "" [PortName "TCAM_I_PORTS", PortName "TCAM_DONE",
                                        PortName "TCAM_MISS", PortName "TCAM_HIT"],
                        -- PortName "TCAM_RPY",
                        PortName "REQ_ENB",
                        PortName "RPY_ENB"
                       ]
        ]
    , t_output  = PortProduct "" [PortProduct "" [PortName "O_DATA", PortName "O_KEEP", PortName "O_USER",
                                                  PortName "O_VALID", PortName "O_READY", PortName "O_LAST"],
                                  PortProduct "" [PortName "TCAM_O_DST_MAC", PortName "TCAM_O_SRC_MAC",
                                                  PortName "TCAM_O_SRC_PORT", PortName "TCAM_O_LOOKUP_REQ"]
                                  -- PortName "TCAM_REQ"
                                  ]
    }) #-}

topEntity   
  :: Clock System Source
  -> Reset System Asynchronous
  -> Signal System (Stream, TCAMreply, Bit, Bit)
  -> Signal System (Stream, TCAMrequest)

topEntity = exposeClockReset (mealy opl_pass_mealy oplInitalState)
{% endhighlight %}

# Footnotes

[^1]: The bandwidth of our switch is limited by the mapping lookup rate and this latency. As it tuns out, we could not add more ports as we are limited in both latency and bandwidth to the mapping module. The mapper is fully scheduled with no free clock lookups. Simultainusly the OPL is the same length as the width of the output port MAC. If it took 9 clocks to identify the destination, the first 64B of the packet would have to be delivered to the MAC without a destination.