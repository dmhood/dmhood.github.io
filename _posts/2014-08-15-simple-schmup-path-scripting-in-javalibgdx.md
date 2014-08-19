---
layout: post
title: "Simple Schmup Path Scripting in Java/LibGDX"
description: ""
category: Software
tags: [software,game development]
---
There are many good resources on building your own schmup (shoot-em-up) for Android, iOS, etc., but I haven't seen too many code samples out there than can be used for basic AI scripts (at least for LibGDX).  For my game, I used a pretty standard entity hierarchy when describing all of my game objects.  In this post I will be focusing on the enemy ships and their scripts, but the logic can really be applied to any type game logic.  Note that all the examples were created on top of the great <a href="http://www.libgdx.com">LibGDX</a> engine.

Each enemy has quite a few variables to manage state, but the ones we are interested in (which deal with movement logic) are as follows:

    {% highlight java linenos=table %}
    /*First, some basic state variables*/

    // time entity has been alive
    private float iTime;
    // The current state of the logic (what step we are in)
    private int iState;

    /*a set of vector points that can hold a flight path.  These are only
      created once per enemy (and then pooled/re-used), so we aren't always
      creating new vector objects during gameplay.*/
    Vector2[] cp = {
      new Vector2(0,0),
      new Vector2(0,0),
      new Vector2(0,0),
      new Vector2(0,0),
      new Vector2(0,0),
      new Vector2(0,0),
      new Vector2(0,0),
      new Vector2(0,0),
    }

    /*we will use the points above to build a path, if necessary*/
    CatmullRomSpline this.path = new CatmullRomSpline<Vector2>(cp, false);

    // The overall logic state to use
    private AIMovementScript iLogic;
    //a point the enemy usually chases
    private Vector2 pathTarget;
    {% endhighlight %}

Whenever an enemy is created, it is assigned a specific movement script/movement logic.  This logic usually takes two forms, either:

1.)  Dynamic logic using state machines.  Essentially just using variables such as iState and iTime to transition the enemy into different modes.  For example: chase the player for 10 seconds-> fly in a circle for the next 10 seconds-> run away.

2.)  Static logic using pre-computed paths.  Other scripts set a variety of screen coordinates and compute the corresponding CatmullRomSpline.  Enemies then follow this path.

In each enemy’s update function, we execute a script enum that has been assigned at build time, and update the ship’s new position/velocity/etc.  I found enums handy to use for all of my scripting, but there are probably several ways to implement different logic scripts.  A simplified update function looks like this:

  {% highlight java linenos=table %}
    public void update() {
      /*use static vector objects (desiredVelocity and actualVector) so we can
      avoid running the garbage collector*/
      desiredVelocity = this.getiLogic.execute(this); //execute the movement script

      /*do whatever else needs updating, like firing at the player, etc.*/

      /*The movement functions only calculate the "desired" velocity for the
      enemy, now we take into account the enemy's mass and current trajectory to
      get our actual velocity*/
      if (desiredVelocity.x == 0 || desiredVelocity.y == 0 ) {
        /*edge case where an enemy is set to fly in a straight line (otherwise
          these are never exactly 0)*/
        actualVelocity.set(desiredVelocity);
      } else {
        //scale the velocity set my our movement script by the enemy's mass
        desiredVelocity.scl(1 / this.mass);
        //This is our drag
        actualVelocity.set(this.lastVelocity);
        //now we have the correct current velocity
        actualVelocity.add(desiredVelocity);
      }
      //normalize the vector so that the enemy's speed setting is consistent
      actualVector.nor();
      this.velocity.set(actualVelocity);
      this.getPosition().add(this.velocity.scl(Gdx.graphics.getDeltaTime()
  								* this.getSPEED())));
      //save this for the next frame
      this.lastVelocity.set(this.velocity);

      /*do any other updating....*/
    }
{% endhighlight %}

So what do the scripts actually look like? First lets get a helper function to chase a single point:
{% highlight java linenos=table %}

    /**
    * Basic chase logic. Set the entity to follow whatever point we pass in.
    *
    * @param e
    * @param point
    **/
    private static void chasePoint(Enemy e, Vector2 point) {
      // Make sure it moves smoothly, instead of just
      // always either diag or straight, which sucks
      floatxdif = Math.abs(e.getPosition().x - point.x);
      floatydif = Math.abs(e.getPosition().y - point.y);

      if (xdif > ydif) {
        e.setVelocity(2, 1);
      }

      if (ydif > xdif) {
        e.setVelocity(1, 2);
      }

      if (xdif == 0) {
        e.setVelocity(0, 3);
      }

      if (ydif == 0) {
        e.setVelocity(3, 0);
      }

      if (xdif == ydif) {
        e.setVelocity(1, 1);
      }

      if (e.getPosition().x > point.x)
        e.getVelocity().x *= -1;

      if (e.getPosition().y > point.y)
        e.getVelocity().y *= -1;
    }
{% endhighlight %}


Finally, here is the enum interface and a couple of basic scripts.  A great plus of this setup is that it is easy to have scripts that are combinations of others.
{% highlight java linenos=table %}

    /**
    * We use an interface in case we one day need something outside of the base
    * class.
    *
    * @author Daniel
    *
    */
    public interface AIMovementScript {
      public void execute(Enemy e);
    }

    private enum EnemyAIMovementScript implements AIMovementScript {
      /*
       * Move the entity using CHASE logic
       **/
       LOGIC_CHASE_TARGET {
         @Override
         public void execute(Enemy e) {
           chasePoint(e, e.getTarget().getPosition());
         }
       },

      /**
      *Set the enemy to chase his target, then retreat when he gets close
      */
      RUSH_THEN_RETREAT {
        @Override
        public void execute(Enemy e) {
          float xdif = Math.abs(e.getPosition().x
            - e.getTarget().getPosition().x);
          float ydif = Math.abs(e.getPosition().y
          	- e.getTarget().getPosition().y);
          double iDif = Math.sqrt(xdif * xdif + ydif * ydif);

          if (e.getiState() == 0) {
            AIHelperMovementScript.LOGIC_CHASE_TARGET.execute(e);
            if (iDif < 10) {
              e.setiState(1);
            }
          } else {
            /*This is just the opposite of the chase logic*/
            AIHelperMovementScript.LOGIC_RETREAT.execute(e);
          }
        }
      },

      /**
      *Set the enemy to follow a basic sine curve
      */
      SINE_CURVE_MIDDLE {
        @Override
        public void execute(Enemy e) {
          if (e.getiState() == 0) {
            Vector2 cp[] = e.getPath().controlPoints;
            cp[0].set(w, h/2);
            cp[1].set(w, h/2);
            cp[2].set(3 * w / 4, h *3 /2);
            cp[3].set(w / 2, h/2);
            cp[4].set(w / 4, h /2);
            cp[5].set(w / 4, h /4);
            cp[6].set(0, h/2);
            cp[7].set(0, 0);
            buildAndSetPath(e, cp);
          }
          if (e.getiState() == 1) {
            /* This logic just sets the enemys target along the path above,
            keeping it just out of reach.  The enemy then just chases this point
            along the path.*/
            AIHelperMovementScript.LOGIC_FOLLOW_PATH.execute(e);
            if (e.isAtEndOfPath()) {
              e.setiState(2);
            }
          }
          if (e.getiState() == 2) {
            //once we are done with the curve just move in a straight line
            EnemyShipMovementScript.STRAIGHT_LINE_MOVEMENT_LEFT
              .execute(e);
          }
        }
      },
    }
{% endhighlight %}

These are just a few of the necessary scripts and helper functions that I'm using, but they all follow the same general pattern.  Another benefit of this method is that you can easily move away from a massive enemy or entity object and keep the movement state/logic as a mixin (that can be applied to essentially anything).
