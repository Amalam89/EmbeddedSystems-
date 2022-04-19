/*
 * Alarm clock with DST, 3 timezones and stopwatch which saves laptimes
 *
 * Created: 3/4/2022 8:31:06 AM
 * Author : alima
 */ 

#include <avr/io.h>
#include "avr.h"
#include "lcd.h"



int state = 0;
int alarm_state = 0;
int alarm_number = 0;
int total_alarms = 0;
int stop_watch_state = 0;
int global_region = 0;


int date[16];


/*Keypad function*/
int is_pressed(int r, int c){
	/* all 8 GPIOs to N/C */
	DDRC = 0;
	PORTC = 0;
	/*set r to "0" */
	CLR_BIT(PORTC,r);
	SET_BIT(DDRC,r);
	/*set c to "w1"*/
	SET_BIT(PORTC,c+4);
	CLR_BIT(DDRC,c+4);
	
	avr_wait(1);
	if(GET_BIT(PINC,c+4)==0/*---_GPIO @ C is "0"*/){
		return 1;
	}
	return 0;
}

/*Keypad function*/
int get_key(){
	int i,j;
	for(i=0;i<4;++i){
		for(j=0;j<4;++j){
			if(is_pressed(i,j)){
				return i*4+j+1;
			}
		}
	}
	return 0;
}

/*structure for clock date and time*/
struct dt{
	int year;
	int month;
	int day;
	int hour;
	int minute;
	int second;
};

/*structure for alarm time*/
struct alarm{
	int minute;
	int hour;
	};

/*structure for lap time*/
struct lap_time{
	int second;
	int minute;
	int hour;
};


/*Initialization for reseting clock*/
void init_dt(struct dt *dt, struct dt *tmpdt){
	dt->year = 2022;
	dt->month = 2;
	dt->day = 10;
	dt->hour = 23;
	dt->minute = 59;
	dt->second = 00;
	
	tmpdt->year = 00;
	tmpdt->month = 00;
	tmpdt->day = 00;
	tmpdt->hour = 00;
	tmpdt->minute = 00;
	tmpdt->second = 00;
}

/*Initialization for reseting alarm*/
void init_alarm(struct alarm *new_alarm){
	new_alarm->hour = 00;
	new_alarm->minute = 00;
}

/*Initialization for reseting lap time*/
void init_lap_time(struct lap_time *lap_time){
	lap_time->second = 00;
	lap_time->minute = 00;
	lap_time->hour = 00;
}

/*print_dt called within main function to constantly display current time and date*/
void print_dt(const struct dt *dt){
	char buf[17];
	lcd_pos(0,0);
	sprintf(buf, "%02d / %02d / %04d",
	dt->month,
	dt->day,
	dt->year);
	lcd_puts2(buf);
	
	lcd_pos(1,0);
	sprintf(buf, "%02d : %02d : %02d",
	dt->hour,
	dt->minute,
	dt->second);
	lcd_puts2(buf);
}

void print_lap_time(const struct lap_time *laptime){
	char buf[17];	
	
	lcd_pos(1,0);
	sprintf(buf, "%02d : %02d : %02d",
	laptime->hour,
	laptime->minute,
	laptime->second);
	lcd_puts2(buf);
}

/*Used primarily to debug the time and date setting mode*/
void blink(int r, int c, const struct dt *dt) {
	char buf[17];
	lcd_pos(r, c);
	if( 0 == r  ) {
		if (c==10){
			lcd_puts2("    ");
			avr_wait(250);
			lcd_pos(0, 0);
			sprintf(buf, "%02d / %02d / %04d",
			dt->month,
			dt->day,
			dt->year);
			lcd_puts2(buf);
		avr_wait(250);}
		else{
			lcd_puts2("  ");
			avr_wait(250);
			lcd_pos(0, 0);
			sprintf(buf, "%02d / %02d / %04d",
			dt->month,
			dt->day,
			dt->year);
			lcd_puts2(buf);
		avr_wait(250);}
	}
	
	if( 1 == r ) {
		lcd_puts2("  ");
		avr_wait(250);
		lcd_pos(1, 0);
		sprintf(buf, "%02d : %02d : %02d",
		dt->hour,
		dt->minute,
		dt->second);
		lcd_puts2(buf);
		avr_wait(250);
	}
}
/*Blink Alarm does not work till I change this comment!*/
void blink_alarm(int c, const struct alarm *new_alarm){
	char buf[17];
	lcd_pos(1, c);
	lcd_puts2("  ");
	avr_wait(250);
	lcd_pos(1, 0);
	sprintf(buf, "Alarm Set:%d:%d",
	new_alarm->hour,
	new_alarm->minute);
	lcd_puts2(buf);
	avr_wait(250);
	
}


/*Use check_alarm in advance_dt function to trigger when time and alarm coincide????*/
void check_alarm(struct dt *dt, struct alarm ALARMS[]){
	int i;	
	for(i=0;i<total_alarms;i++){
		if (ALARMS[i].hour==dt->hour && ALARMS[i].minute==dt->minute){							
			ring_alarm();		
			}		
		}
	}	

/*Sound off alarm*/
void ring_alarm(){
	
	int i, j;
	int k = 600;
	SET_BIT(DDRB,3);	
	
	for(j=0;j<3;j++){	
		for(i=0;i<k;++i){
			SET_BIT(PORTB,3);
			avr_wait(1);
			CLR_BIT(PORTB,3);
			avr_wait(1);
			}
		avr_wait(100);
	}
}
/*Logic for time and date cascading ie. sec->min->hour->day->month->year????*/
void advance_dt(struct dt *dt,struct alarm ALARMS[]){
	++dt->second;
	if(60==dt->second){
		dt->second = 0;
		++dt->minute;						/*this was a case where alarm was orginally going off on every minute of 00:00 */		
		check_alarm(dt,ALARMS);		/*due to the fact of that upon sec reaching 60 it would interally become 00:00:00*/
		if(60==dt->minute){					/*which would retrigger the alarm incorrectly;*/
			dt->minute = 0;
			++dt->hour;						/*need to find appropriate location to trigger alarm*/
			check_alarm(dt,ALARMS);
			if(24==dt->hour){
				dt->hour = 0;
				check_alarm(dt,ALARMS);
				if(31==dt->day && ((1==dt->month)||(3==dt->month)||(5==dt->month)||(7==dt->month)||(8==dt->month)||(10==dt->month))){
					dt->day = 1;
				++dt->month;}
				else if(31==dt->day && (12==dt->month)){
					dt->day = 1;
					dt->month = 1;
				++dt->year;}
				else if(28==dt->day && (2==dt->month)){
					dt->day = 1;
				++dt->month;}
				else if(30==dt->day && ((4==dt->month)||(6==dt->month)||(9==dt->month)||11==dt->month)){
					dt->day = 1;
				++dt->month;}
				else{
					++dt->day;
				}
			}
		}
	}
}




void advance_stop_watch(struct lap_time *lap_time){
	++lap_time->second;
	if(60==lap_time->second){
		lap_time->second = 0;
		++lap_time->minute;		
		if(60==lap_time->minute){
			lap_time->minute = 0;
			++lap_time->hour;
			}
		}
	}

/*stop watch mode?????*/
int stop_watch(int e){
	
	char buf[17];		
	int i,k;
	int total_times = 0;
	struct lap_time LAP_TIMES[3];
	struct lap_time current_lap_time;
	init_lap_time(&current_lap_time);
	
	while(1){
		
		lcd_pos(0, 0);
		sprintf(buf, "StopWatch Mode");
		lcd_puts2(buf);
		print_lap_time(&current_lap_time);
		
		k = get_key();
		avr_wait(500);
	
		if(1==k && stop_watch_state==0){
			stop_watch_state = 1;
			/* start stop watch*/
			}
		else if (1==k && stop_watch_state==1){
			stop_watch_state = 0;
			/* pause stop watch*/
			}
		else if (2==k && stop_watch_state==1){
			LAP_TIMES[total_times].second = current_lap_time.second;
			LAP_TIMES[total_times].minute = current_lap_time.minute;
			LAP_TIMES[total_times].hour = current_lap_time.hour;
			lcd_pos(0, 0);
			sprintf(buf, "Saved Lap Time:");
			lcd_puts2(buf);
			print_lap_time(&current_lap_time);
			avr_wait(2000);
			stop_watch_state = 0;
			total_times++;
			init_lap_time(&current_lap_time);	
			lcd_clr();		
			/*save the lap time and reset stop watch, can save up to two lap times*/
			}
		else if (3==k && stop_watch_state==0){
			for(i=0;i<total_times;i++){
				lcd_pos(0, 0);
				sprintf(buf, "Saved Lap Time:%d", i);
				lcd_puts2(buf);
				print_lap_time(&LAP_TIMES[i]);
				avr_wait(2000);
				/*print every laptime with avr_wiat in between*/
				}
			lcd_clr();	
			}
		else if (16==k){
			return 2;
		}
		if(stop_watch_state==1){
			avr_wait(1000);
			advance_stop_watch(&current_lap_time);
			print_lap_time(&current_lap_time);
			}		
		}
	}
	
/*Will use this to add relevant time region, three regions for this program*/
int select_worldtime(int e, struct dt* dt){
	
	char buf[17];
	lcd_pos(0, 0);
	sprintf(buf, "Select Timezone");
	lcd_puts2(buf);	
	lcd_pos(1, 0);
	sprintf(buf, "1PST 2CST 3EST");
	lcd_puts2(buf);
	
	
	
	int k, PST,CST,EST;
	k = get_key();
	avr_wait(500);
	
	if(global_region==0){
		PST=0;
		CST=2;
		EST=3;
		}
	if (global_region==1){
		PST=-2;
		CST=0;
		EST=1;
		}
	if (global_region==2){
		PST=-3;
		CST=-1;
		EST=0;
		}
	
	if(1==k){
		dt->hour = dt->hour+PST;			
		global_region = 0;		
		}
	else if (2==k){
		dt->hour = dt->hour+CST;			
		global_region = 1;		
		}
	else if (3==k){
		dt->hour = dt->hour+EST;		
		global_region = 2;		
		}	
	/*go to next day, month, year if applicable*/
	if(dt->hour>=24){
		dt->hour = abs(dt->hour%24);/*orig dt->hour=0*/
		if((31==dt->day) && ((1==dt->month)||(3==dt->month)||(5==dt->month)||(7==dt->month)||(8==dt->month)||(10==dt->month))){
			dt->day = 1;
		++dt->month;}
		else if(31==dt->day && (12==dt->month)){
			dt->day = 1;
			dt->month = 1;
		++dt->year;}
		else if(28==dt->day && (2==dt->month)){
			dt->day = 1;
		++dt->month;}
		else if(30==dt->day && ((4==dt->month)||(6==dt->month)||(9==dt->month)||11==dt->month)){
			dt->day = 1;
		++dt->month;}
		else{
			++dt->day;
			}
		}
	/*go back to prev day and month and maybe year*/
	else if ((dt->hour<0) && (dt->day==1)){/**/
		/*reversing the clock in the instance where go back in timezone to lead to the previous month etc*/
		
		dt->hour = 24+dt->hour;
		
		if(1==dt->month){
			dt->month = 12;
			dt->day = 31;
			dt->year = dt->year - 1;
			}
		if(2==dt->month){
			dt->month = 1;
			dt->day = 31;
			}
		else if(3==dt->month){
			dt->month = 2;
			dt->day = 28;
			}
		else if (4==dt->month){
			dt->month = 3;
			dt->day = 31;
			}
		else if (5==dt->month){
			dt->month = 4;
			dt->day = 30;
			}
		else if(6==dt->month){
			dt->month = 5;
			dt->day = 31;
			}
		else if(7==dt->month){
			dt->month = 6;
			dt->day = 30;
			}
		else if(8==dt->month){
			dt->month = 7;
			dt->day = 31;
			}
		else if(9==dt->month){
			dt->month = 8;
			dt->day = 31;
			}
		else if(10==dt->month){
			dt->month = 9;
			dt->day = 30;
			}
		else if(11==dt->month){
			dt->month = 10;
			dt->day = 31;
			}
		else if(12==dt->month){
			dt->month = 11;
			dt->day = 30;
			}
		}
	/*go back to prev day*/
	else if(dt->hour<0){
		dt->hour = 24+dt->hour;
		dt->day = dt->day-1;
	}
	/*once all relevant updates to dt are made with valid input return functions*/
	if(k==1||k==2||k==3){
		return 0;
	}
	
	else if (16==k){
		return -1;
	}
	return 1;
}

/*Set alarm mode????*/
int set_alarm(int e, struct alarm* new_alarm, struct alarm ALARMS[]){	
	char buf[17];
	lcd_pos(0, 0);
	sprintf(buf, "Alarm Set Mode");
	lcd_puts2(buf);	
	lcd_pos(1, 0);
	sprintf(buf, "Alarm Set:");
	lcd_puts2(buf);
	
	if(1==e){
		alarm_state++;}	
	

	int k,i;	
	k = get_key();
	avr_wait(500);
	
	
		
	if(16==k){
		return -1;}/*exit command*/
	/*if(1==alarm_state || 2==alarm_state){
		blink_alarm(10,&new_alarm);
	}
	if(3==alarm_state||4==alarm_state){
		blink_alarm(,&new_alarm);}*/
	
	
	if((0<k && 4>k) || (4<k && 8>k) || (8<k && 12>k) || (14==k) ){		
		/*Mapping/converting the keys of the keyboard to the respective values.*/		
		if(4<k && 8>k){
		k = k - 1;}
		if(8<k && 12>k){
		k = k - 2;}
		if(14==k){
		k = 0;}			
		
		
		
		if(1==alarm_state){
			new_alarm->hour = k*10;
			lcd_pos(1, 0);
			sprintf(buf, "Alarm Set:%d :", new_alarm->hour);
			lcd_puts2(buf);
			return 1;			
		}
		if(2==alarm_state){
			new_alarm->hour += k;
			if (new_alarm->hour>23){
				return -1;}			
			lcd_pos(1, 0);
			sprintf(buf, "Alarm Set:%02d:", new_alarm->hour);
			lcd_puts2(buf);
			return 1;
		}
		if(3==alarm_state){
			new_alarm->minute = k*10;
			lcd_pos(1, 0);
			sprintf(buf, "Alarm Set:%02d:%d", new_alarm->hour, new_alarm->minute);
			lcd_puts2(buf);
			return 1;		
		}
		if(4==alarm_state){
			new_alarm->minute += k;
			if (new_alarm->minute>59){
			return -1;}
			lcd_pos(1, 0);
			sprintf(buf, "Alarm Set:%02d:%02d", new_alarm->hour, new_alarm->minute);
			lcd_puts2(buf);
			return 1;
		}
		if(5==alarm_state){			
			ALARMS[alarm_number].hour = new_alarm->hour;
			ALARMS[alarm_number].minute = new_alarm->minute;
			if (alarm_number==4){
				alarm_number = 0;
				}
			else{				
				alarm_number += 1;
				total_alarms += 1;
				}	
			lcd_pos(0, 0);
			sprintf(buf, "Created Alarm %d", alarm_number);
			lcd_puts2(buf);
			lcd_pos(1, 0);
			sprintf(buf, "Alarm Set:%02d:%02d", ALARMS[alarm_number-1].hour,ALARMS[alarm_number-1].minute);
			lcd_puts2(buf);
			avr_wait(2000);
			
			for(i=0;i<total_alarms;i++){
				lcd_pos(1, 0);
				sprintf(buf, "Alarmhour%d: %d: %d",i, ALARMS[i].hour, ALARMS[i].minute);
				lcd_puts2(buf);				
				avr_wait(2000);}
			
					
			return 2;/*completed setting an alarm*/
		}		
	}
}
		
/*Setting time and date mode*/
int change_state(int e, struct dt* dt, struct dt* tmpdt){
	char buf[17];
	
	if(1==e){
	state++;}
	
	int k;
	k = get_key();
	avr_wait(500);
	
	if(16==k){
	return -1;}/*exit command*/
	
	else{
		
		if(1==state || 2==state){
			blink(0,0,tmpdt);
		}
		if(3==state||4==state){
			blink(0,5,tmpdt);
		}
		if(5==state||6==state||7==state||8==state){
			blink(0,10,tmpdt);
		}
		if(9==state||10==state){
			blink(1,0,tmpdt);
		}
		if(11==state||12==state){
			blink(1,5,tmpdt);
		}
		if(13==state||14==state){
			blink(1,10,tmpdt);
		}
		
		
		if((0<k && 4>k) || (4<k && 8>k) || (8<k && 12>k) || (14==k) ){
			/*Mapping the keys of the keyboard to the respective values.*/
			if(4<k && 8>k){
			k = k - 1;}
			if(8<k && 12>k){
			k = k - 2;}
			if(14==k){
			k = 0;}
			
			if(1==state){
				tmpdt->month = k*10;
				lcd_pos(0, 0);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
				
			}
			if(2==state){
				tmpdt->month += k;
				if (tmpdt->month>12){
				return -1;}
				dt->month = tmpdt->month;
				lcd_pos(0, 1);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(3==state){
				tmpdt->day = k*10;
				lcd_pos(0, 5);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(4==state){
				tmpdt->day += k;
				if (tmpdt->day>31
				|| (tmpdt->day>30 && ((4==tmpdt->month)||(6==tmpdt->month)||(9==tmpdt->month)||(11==tmpdt->month)))
				|| (tmpdt->day>28 && (2==tmpdt->month))){
				return -1;}
				dt->day = tmpdt->day;
				lcd_pos(0, 6);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(5==state){
				tmpdt->year = k*1000;
				lcd_pos(0, 10);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(6==state){
				tmpdt->year += k*100;
				lcd_pos(0, 11);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(7==state){
				tmpdt->year += k*10;
				lcd_pos(0, 12);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(8==state){
				tmpdt->year += k;
				if (tmpdt->year>9999){
				return -1;}
				dt->year = tmpdt->year;
				lcd_pos(0, 13);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(9==state){
				tmpdt->hour += k*10;
				lcd_pos(1, 0);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(10==state){
				tmpdt->hour += k;
				if (tmpdt->hour>23){
				return -1;}
				dt->hour = tmpdt->hour;
				lcd_pos(1, 1);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(11==state){
				tmpdt->minute += k*10;
				lcd_pos(1, 5);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(12==state){
				tmpdt->minute += k;
				if (tmpdt->minute>59){
				return -1;}
				dt->minute = tmpdt->minute;
				lcd_pos(1, 6);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(13==state){
				tmpdt->second += k*10;
				lcd_pos(1, 10);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				return 1;
			}
			if(14==state){
				tmpdt->second += k;
				if (tmpdt->second>59){
				return -1;}
				lcd_pos(1, 11);
				sprintf(buf, "%d", k);
				lcd_puts2(buf);
				dt->second = tmpdt->second;
				return 1;
			}
			if(15==state){
				return 2;
			}
		}
	}

	return 0;
}



int main(void){
	/* Replace with your application code */
	int k, e, i;
	struct dt dt;
	struct dt tmpdt;
	struct alarm new_alarm;
	/*Array of alarm, five alarms*/
	struct alarm ALARMS[5];
		
	lcd_init();
	init_dt(&dt, &tmpdt);
	init_alarm(&new_alarm);
	
	for(;;){
		int i,k;
		k = get_key();
		e = 1;
		if (15==k){/*press # to set time and date*/			
			while(1){
				e = change_state(e, &dt,&tmpdt);
				if (-1==e){
					lcd_clr();
					init_dt(&dt,&tmpdt);
					state = 0;
					break;
					}
				if (2==e){
					state = 0;
					break;
					}
				}
			}
		else if(k==4){/*A button selecting relative world time */
			lcd_clr();
			while(1){				
				e = select_worldtime(e,&dt);
				if (-1==e || 0==e){					
					lcd_clr();
					break;
					}				
				}
			}		
		else if(k==8){/*B button setting alarm times*/
			lcd_clr();
			while(1){				
				e = set_alarm(e, &new_alarm, ALARMS);
				if (-1==e){					
					lcd_clr();
					init_alarm(&new_alarm);
					alarm_state = 0;
					break;
					}
				if (2==e){
					alarm_state = 0;
					lcd_clr();
					break;
					}
				}			
			}		
		else if(k==12){/*C button enters stopwatch mode, need 2 lap-times to save*/
			lcd_clr();
			while(1){
				e = stop_watch(e);
				
				if (2==e){
					stop_watch_state = 0;
					lcd_clr();
					break;
					}
				}
			}
		else if(k==16){/*D button tests beep for alarm */			
				ring_alarm();
				}
		avr_wait(1000);
		advance_dt(&dt, ALARMS);		
		print_dt(&dt);
		
	}
}


